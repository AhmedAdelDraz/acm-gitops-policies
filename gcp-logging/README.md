# GCP logging (`openshift-gcp-logging`)

Forwards application, infrastructure and audit logs to **Google Cloud
Logging**, authenticating with **GCP Workload Identity Federation** (WIF), so no
service account keys are stored on the cluster.

**Targets** clusters labelled `vendor=OpenShift` and `cloud=Google`. Supported
only on OpenShift on GCP **installed with WIF** (Cloud Credential Operator in
Manual mode). Do not place it on a key-based GCP cluster.

| Policy | Mode | What it does |
|---|---|---|
| `day2-logging-operators` | enforce | Namespaces `openshift-logging` and `openshift-cluster-observability-operator`; Cluster Logging Operator (Logging 6.5 or later is required for WIF) and Cluster Observability Operator. |
| `day2-logging-gcp-forwarder` | enforce | Depends on the policy above. ServiceAccount `logcollector` with the `collect-application/infrastructure/audit-logs` roles; a CredentialsRequest that makes the CCO create `openshift-logging/gcp-wif-secret`; a `ClusterLogForwarder` sending all three log types to Cloud Logging, with a log ID per cluster. |

## How authentication works

The collector's projected ServiceAccount token (audience `openshift`) is
exchanged with GCP STS for a token impersonating the GCP service account
`openshift-logging-writer@<project ID>.iam.gserviceaccount.com`. Everything
cluster specific is discovered on the managed cluster:

- **WIF audience** (project number, pool, provider): from the CCO-managed
  `openshift-ingress-operator/cloud-credentials` secret;
- **GCP project ID:** from `Infrastructure/cluster`.

## Per-cluster prerequisite (one time)

In Manual mode the CCO cannot create GCP service accounts or IAM bindings.
Create them with `gcloud`; the exact commands are in the header of
`day2-logging-gcp-forwarder.yaml`. In short:

1. Create `openshift-logging-writer` in the cluster's project (once per
   project).
2. Grant it `roles/logging.logWriter`.
3. Grant `roles/iam.workloadIdentityUser` on it to
   `principal://iam.googleapis.com/<pool>/subject/system:serviceaccount:openshift-logging:logcollector`.

Clusters sharing a project share the service account; each cluster only adds
its own workloadIdentityUser binding.

**Verify:**

```
oc get clusterlogforwarder gcp-logging -n openshift-logging -o jsonpath='{.status.conditions}'
oc get pods -n openshift-logging
```

Logs then appear in Cloud Logging (Logs Explorer) under the cluster's log ID.

## Current setup

Not deployed on the hub, which is not on GCP (`cloud=Other`). The group is
validated for structure and server-side admission only.

## Required changes for 3 infra nodes

1. **The collector must run on the infra nodes too**, or their logs (routers,
   monitoring, registry) are not forwarded. Check after tainting:
   ```
   oc get ds -n openshift-logging        # DESIRED = number of nodes
   ```
   If the infra nodes are missing, add to the `ClusterLogForwarder` in
   `day2-logging-gcp-forwarder.yaml`:
   ```yaml
   spec:
     collector:
       tolerations:
         - key: node-role.kubernetes.io/infra
           operator: Exists
   ```
2. **Optional:** with more nodes, log volume and Cloud Logging cost grow. Use
   ClusterLogForwarder `filters` to drop noisy namespaces or debug-level logs.

Nothing else depends on the node layout. The collector is a DaemonSet with no
storage, and Cloud Logging is the store.
