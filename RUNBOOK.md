# GPU Monitoring on OpenShift: Runbook

Installs the Grafana Operator, connects Grafana to OpenShift's Thanos Querier, and deploys four GPU dashboards built on NVIDIA DCGM metrics. Works on connected and disconnected clusters, and on fleets that mix MIG-partitioned and whole GPUs.

The whole install is five commands. The checks around them are plain `oc` commands; nothing needs a script or extra tools.

## Contents

1. [Requirements](#1-requirements)
2. [Check your cluster](#2-check-your-cluster-read-only)
3. [Edit the values for your environment](#3-edit-the-values-for-your-environment)
4. [Mirror the operator (disconnected only)](#4-mirror-the-operator-disconnected-clusters-only)
5. [Install](#5-install)
6. [Verify](#6-verify-read-only)
7. [Changing a dashboard](#7-changing-a-dashboard)
8. [Adapting to your environment](#8-adapting-to-your-environment)
9. [Troubleshooting](#9-troubleshooting)
10. [Uninstall](#10-uninstall)
11. [Metric reference](#11-metric-reference)

---

## 1. Requirements

| Item | Notes |
|---|---|
| OpenShift 4.12+ | Tested on 4.18. Uses the built-in monitoring stack (Prometheus + Thanos Querier). |
| NVIDIA GPU Operator | Tested with v25.3. Its DCGM exporter must be scraped by cluster monitoring. |
| DCGM profiling metrics | `DCGM_FI_PROF_GR_ENGINE_ACTIVE` and `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE`. The usual `DCGM_FI_DEV_GPU_UTIL` does not work on MIG instances, so these dashboards don't use it. |
| Grafana Operator v5 | From the community catalog. Tested with v5.24.0. |
| Grafana | Tested with 13.0.1. |
| A StorageClass | Any RWO class, for a 5Gi Grafana volume. |
| Access | cluster-admin for the install. |

## 2. Check your cluster (read-only)

These commands change nothing. Run them first; they tell you whether the dashboards will have data.

Set up a short alias that queries Prometheus from inside its own pod:

```bash
promq() {
  oc -n openshift-monitoring exec -c prometheus prometheus-k8s-0 -- \
    curl -s --get --data-urlencode "query=$1" http://localhost:9090/api/v1/query; echo
}
```

**GPU Operator and exporter are running**

```bash
oc get clusterpolicy
oc get pods -A -l app=nvidia-dcgm-exporter
```

Expect a ClusterPolicy with state `ready` and one Running exporter pod per GPU node.

**Core metrics reach Prometheus**

```bash
promq 'count(count by (UUID) (DCGM_FI_DEV_FB_USED))'      # physical GPUs
promq 'count(DCGM_FI_DEV_FB_USED)'                        # devices (MIG instances count separately)
promq 'count by (Hostname) (DCGM_FI_PROF_GR_ENGINE_ACTIVE)'
promq 'count by (Hostname) (DCGM_FI_PROF_PIPE_TENSOR_ACTIVE)'
```

Each should return numbers in `"result"`. An empty result (`"result":[]`) means:

- No `DCGM_FI_DEV_FB_USED`: the exporter isn't being scraped. Check Observe > Targets in the console for `dcgm`.
- No `DCGM_FI_PROF_*`: profiling metrics are off. Add them to the DCGM exporter's metrics ConfigMap and set `ClusterPolicy.spec.dcgmExporter.config.name` to it. This restarts the exporter DaemonSet, so treat it as a change window on production.

**Which labels your cluster uses**

```bash
promq 'count by (exported_pod) (DCGM_FI_DEV_FB_USED{exported_pod!=""})'
promq 'count by (GPU_I_PROFILE) (DCGM_FI_DEV_FB_USED)'
promq 'count(kube_node_labels{label_nvidia_com_mig_config!=""})'
```

- The first one returning rows means you have the default `exported_pod` / `exported_namespace` labels. If it's empty while GPU workloads are running, see section 8, *Workload label names*.
- The second lists your MIG profiles. An empty `GPU_I_PROFILE` means MIG is disabled on those GPUs, which is fine.
- If the third is empty, the *MIG strategy* columns will be blank. Everything else still works.

**Optional metric**

```bash
promq 'count(gpu_xid_events_total)'
```

If this is empty, see section 8, *gpu_xid_events_total*.

**Cluster version** (needed for mirroring)

```bash
oc get clusterversion version -o jsonpath='{.status.desired.version}'; echo
```

## 3. Edit the values for your environment

```bash
grep -rn CHANGE_ME .
```

| File | What to set |
|---|---|
| `01-operator/operator.yaml` | `source`: `community-operators` on connected clusters, or the CatalogSource oc-mirror created |
| `02-grafana/02-grafana.yaml` | `storageClassName`, route `host`, and `image` (your mirror registry on disconnected clusters) |
| `mirror/imageset-config.yaml` | catalog tag (your OCP minor version) and target registry |

Everything installs into the `gpu-monitoring` namespace. To use another one:

```bash
grep -rl 'gpu-monitoring' 01-operator 02-grafana 03-dashboards | xargs sed -i 's/namespace: gpu-monitoring/namespace: YOUR_NS/; s/^  name: gpu-monitoring$/  name: YOUR_NS/; s/- gpu-monitoring$/- YOUR_NS/'
```

## 4. Mirror the operator (disconnected clusters only)

Skip this on a connected cluster.

On the mirror host (needs `registry.redhat.io`, `ghcr.io` and `docker.io`):

```bash
oc mirror -c mirror/imageset-config.yaml \
  --workspace file:///opt/mirror/grafana \
  docker://quay.ocp.lan:8443 --v2
```

If the mirror host can't reach your registry directly, mirror to disk first (`file://`), carry the folder over, then push with `--from`.

Apply what oc-mirror generated and wait for the catalog:

```bash
oc apply -f /opt/mirror/grafana/working-dir/cluster-resources/
oc get catalogsource -n openshift-marketplace          # wait for READY
oc get packagemanifests | grep grafana-operator
```

Put the CatalogSource name into `01-operator/operator.yaml` as `source`.

## 5. Install

**These steps change the cluster.**

**Step 1: operator**

```bash
oc apply -f 01-operator/
oc get installplan -n gpu-monitoring
oc patch installplan <name> -n gpu-monitoring --type merge -p '{"spec":{"approved":true}}'
oc get csv -n gpu-monitoring -w          # wait for Succeeded, then Ctrl+C
```

Approval is manual on purpose. With automatic approval, a disconnected cluster can try to upgrade to a version that was never mirrored, and the subscription gets stuck.

**Step 2: admin password**

```bash
oc -n gpu-monitoring create secret generic grafana-admin-credentials \
  --from-literal=GF_SECURITY_ADMIN_USER=admin \
  --from-literal=GF_SECURITY_ADMIN_PASSWORD="$(openssl rand -base64 24)"
```

This Secret is created by command, not kept in a file, so the password never ends up in Git.

**Step 3: Grafana, its service account and the datasource**

```bash
oc apply -f 02-grafana/
oc -n gpu-monitoring get pods -w          # wait for the grafana pod to be Running
```

**Step 4: dashboards**

```bash
oc apply -f 03-dashboards/
```

The four dashboards link to each other, so install all four.

| Dashboard | For |
|---|---|
| GPU Fleet Summary | Management: one screen of usage, memory, power, errors, and who uses what |
| GPU Fleet Telemetry | Operators: per-device inventory, health, MIG layout, idle capacity |
| GPU Utilisation by Application (MIG-aware) | Chargeback: usage per namespace and pod, weighted by MIG slice size |
| GPU XID Errors | Which GPUs report XID faults |

## 6. Verify (read-only)

```bash
oc -n gpu-monitoring get csv
oc -n gpu-monitoring get pods
oc -n gpu-monitoring get grafana,grafanadatasource,grafanadashboard
oc -n gpu-monitoring describe grafana gpu-grafana | grep -A3 -i 'status\|message'
```

Look for: the CSV in `Succeeded`, the grafana pod `Running`, and no `failed to authenticate` message on the Grafana resource (section 9 if there is one).

Get the URL and the password:

```bash
oc -n gpu-monitoring get route
oc -n gpu-monitoring get secret grafana-admin-credentials \
  -o jsonpath='{.data.GF_SECURITY_ADMIN_PASSWORD}' | base64 -d; echo
```

Log in and check:

- Fleet Summary: **Total GPUs Installed** matches the physical GPU count from section 2.
- Fleet Telemetry: **Total devices** matches the device count from section 2.
- Fleet Telemetry: the **Fleet baseboard** table lists every node.

## 7. Changing a dashboard

The YAML files in `03-dashboards/` are the source of truth. **Edits made only in the Grafana UI don't last:** the operator puts the file's version back on its next resync. To keep a change:

1. Edit the dashboard in Grafana and save.
2. Click **Export > Export as JSON**. Choose the **Classic** format and turn on **Export for sharing externally**. Copy the JSON.
   - Check the result: it must have a top-level `"panels"` list. If you see `"elements"` and `"layout"` instead, that's the newer format, and the operator won't load it.
3. Open the matching file in `03-dashboards/` and replace everything under `json: |` with the copied JSON, indented 4 spaces. In most editors you can paste, keep the block selected, and press Tab twice.
4. Leave `"uid"` unchanged. The dashboards link to each other by UID.
5. Apply and commit:
   ```bash
   oc diff -f 03-dashboards/<file>.yaml
   oc apply -f 03-dashboards/<file>.yaml
   ```

If you rename a table column, check that panel's overrides too. Overrides match columns by name, so a renamed column quietly loses its units and decimals.

## 8. Adapting to your environment

### Workload label names

On OpenShift, Prometheus renames the exporter's `pod` and `namespace` labels to `exported_pod` and `exported_namespace`. The Telemetry and MIG-aware dashboards have **Pod label** and **Namespace label** dropdowns for clusters that keep the raw names.

The Fleet Summary panels, and **Node capacity** on Telemetry, use `exported_pod` / `exported_namespace` directly. If your cluster uses plain `pod` / `namespace`:

```bash
sed -i 's/exported_pod/pod/g; s/exported_namespace/namespace/g' \
  03-dashboards/gpu-fleet-summary.yaml 03-dashboards/gpu-fleet-telemetry.yaml
oc apply -f 03-dashboards/
```

### MIG profile names

DCGM exporter names MIG profiles from the usable memory of each slice, which is a little less than NVIDIA's official name. On H200 141GB:

| DCGM label | Official NVIDIA name |
|---|---|
| `1g.16gb` | `1g.18gb` |
| `2g.33gb` | `2g.35gb` |
| `3g.70gb` | `3g.71gb` |
| `7g.140gb` | `7g.141gb` |

The queries use the DCGM names. The **MIG profile** columns only relabel them for display, using value mappings in `gpu-fleet-telemetry.yaml` and `gpu-utilisation-by-application.yaml`. These mappings are written for H200. On other GPUs, compare `nvidia-smi mig -lgi` (run in the driver pod) with the second query in section 2, then edit those mappings or delete them to show the raw names.

### Slice weighting

GPU-equivalent panels weight each MIG instance by its compute slices out of 7, matching on the slice prefix (`GPU_I_PROFILE=~"3g.*"` counts 3/7). That holds for A100, H100, H200 and B200, which all have 7 compute slices. GPUs with MIG disabled count as 1.

### gpu_xid_events_total

**Total GPU Errors** on Fleet Summary and the **GPU XID Errors** dashboard read `gpu_xid_events_total`. DCGM exporter doesn't produce this metric, and this repo doesn't create it. Without it those panels show *No data*; everything else works. **Health events** on Fleet Telemetry uses DCGM's own `DCGM_FI_DEV_XID_ERRORS` and doesn't need it.

### Timeseries panels on Grafana 13.0.1

On Grafana 13.0.1 we saw timeseries panels fail to render even with correct data, so the trend panels on Fleet Summary are bar charts. Five timeseries panels remain: four on the MIG-aware dashboard and **Active GPU equivalents** on Fleet Telemetry. If they're blank on your version while the same query works in Explore, switch them to bar charts.

## 9. Troubleshooting

| Symptom | Cause and fix |
|---|---|
| Grafana resource says `failed to authenticate with grafana` | The operator can't log in to Grafana. Usually someone changed the admin password in the Grafana UI, so it no longer matches `grafana-admin-credentials`. Put the live password into the Secret (or reset Grafana's to the Secret's), then delete the grafana pod. Never change the admin password from the UI. |
| Admin password is random, not the one in the Secret | `disableDefaultAdminSecret: true` is missing from `02-grafana/02-grafana.yaml`. |
| Logged in fine, but no dashboards appear | Same cause as the first row. Also check that the Grafana CR uses `env` with `secretKeyRef`, not `envFrom`; the operator can only read credentials from `env`. |
| Dashboards revert after editing in the UI | Expected. Follow section 7. |
| GrafanaDashboard stuck deleting | Its finalizer waits for the operator to remove the dashboard from Grafana, which fails while authentication is broken. Fix authentication first. As a last resort: `oc patch grafanadashboard <name> -n gpu-monitoring --type merge -p '{"metadata":{"finalizers":null}}'` |
| Every panel shows *No data* | Datasource problem. Check `oc get clusterrolebinding grafana-thanos-reader-monitoring-view` exists and that the datasource URL uses port **9091**. Port 9092 rejects cross-namespace queries. |
| Some panels empty, others fine | A metric is missing. Run the section 2 checks. |
| Utilisation is 0 on MIG nodes | Profiling metrics are off (section 2). |
| Grafana pod stuck in `ContainerCreating` after a change | The volume is still attached to the old pod. The CR uses the `Recreate` strategy to prevent this; if someone changed it, delete the old pod. |
| `ImagePullBackOff` | Disconnected cluster: an image wasn't mirrored, or the IDMS from oc-mirror wasn't applied. The operator comes from `ghcr.io`, Grafana from `docker.io`. |
| Garbled output from long `oc exec ... curl` commands | Large output over `oc exec` can arrive corrupted on some networks. Ask narrow questions (like the `count by` queries in section 2) so little data comes back. What's stored in Prometheus is not affected. |

## 10. Uninstall

**Changes the cluster.** Remove in reverse order, so the operator can clean up the dashboards while it's still running:

```bash
oc delete -f 03-dashboards/
oc delete -f 02-grafana/                  # also deletes the PVC and Grafana's data
oc -n gpu-monitoring delete secret grafana-admin-credentials
oc delete -f 01-operator/                 # removes the namespace too
```

The Grafana CRDs stay on the cluster. Remove them only if nothing else uses the Grafana Operator.

## 11. Metric reference

| Metric | Used for |
|---|---|
| `DCGM_FI_PROF_GR_ENGINE_ACTIVE` | Utilisation (works per MIG instance) |
| `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` | Tensor core activity |
| `DCGM_FI_DEV_FB_USED`, `DCGM_FI_DEV_FB_FREE` | GPU memory and device counts (MiB, per MIG instance) |
| `DCGM_FI_DEV_POWER_USAGE` | Power (device level, de-duplicated by `UUID`) |
| `DCGM_FI_DEV_TOTAL_ENERGY_CONSUMPTION` | Energy (mJ counter, de-duplicated by `UUID`) |
| `DCGM_FI_DEV_GPU_TEMP` | Temperature |
| `DCGM_FI_DEV_XID_ERRORS` | Health events table |
| `kube_node_labels` | MIG strategy per node (`label_nvidia_com_mig_config`) |
| `kube_node_status_allocatable` | Advertised GPUs per node |
| `gpu_xid_events_total` | XID counts (optional, not provided here) |

How the queries count:

- **Physical GPUs:** `count(count by (UUID) (...))`. MIG instances share their card's UUID.
- **Devices:** one series per MIG instance or whole GPU. Grouping also uses `GPU_I_ID`, because two same-profile slices on one card would otherwise merge into one row.
- **Power and energy:** device-level values repeated on every MIG series, so they're averaged by `UUID` before summing.
