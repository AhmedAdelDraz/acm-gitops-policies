# Network Observability (`openshift-network-observability`)

Network flow monitoring with the Network Observability Operator: an eBPF agent
on every node, flows stored in Loki, dashboards and a topology view in the web
console (**Observe → Network Traffic**).

**Opt-in per cluster:**

```
oc label managedcluster <cluster> network-observability=enabled
```

Loki needs object storage. On **OpenShift on GCP installed with Workload
Identity Federation** it uses a **Google Cloud Storage** bucket (see
[GCS on GCP](#gcs-on-gcp)); on every other cluster it uses the Multicloud
Object Gateway (`openshift-storage.noobaa.io` StorageClass, part of ODF). The
cluster also needs a default StorageClass.
(`openshift-storage.noobaa.io` StorageClass, part of ODF) and a default
StorageClass.

| Policy | Mode | What it does |
|---|---|---|
| `day2-netobserv-operators` | enforce | On GCP with WIF, the Loki Operator gets the WIF pool, provider, project number and GCP service account as environment variables. Namespaces `openshift-operators-redhat`, `openshift-netobserv-operator`, `netobserv`. **Loki Operator first** (channel `stable-6.6`), then the Network Observability Operator (channel `stable`). Both channels are pinned because `community-operators` also ships `loki-operator`. |
| `day2-netobserv-flowcollector` | enforce + inform checks | Depends on the policy above. Steps, in order (see below). |

`day2-netobserv-flowcollector` steps:

1. `netobserv-operator-rbac`: RoleBindings `secret-watcher` and
   `secret-creator` for the operator in `netobserv`. The operator only creates
   them when it creates the namespace itself.
2. `netobserv-loki-bucket`: ObjectBucketClaim `loki-netobserv` on MCG, and the
   ConfigMap `loki-s3-ca`, into which the service CA is injected.
3. `netobserv-loki-bucket-ready` (inform): the claim is `Bound`.
4. `netobserv-loki-s3-secret`: Secret `loki-s3`, built on the cluster from the
   claim's generated credentials (bucket, `https://s3.openshift-storage.svc` endpoint).
5. `netobserv-lokistack`: LokiStack `loki`, size **`1x.demo`**, schema v13,
   tenant mode `openshift-network`, PVCs on the default StorageClass.
6. `netobserv-lokistack-ready` (inform): LokiStack `Ready`.
7. `netobserv-flowcollector`: FlowCollector `cluster`, `deploymentModel: Direct`
   (no Kafka), eBPF agent **`privileged: true`**, sampling 1:50, features
   **`PacketDrop`, `DNSTracking`, `FlowRTT`**, console plugin on.
8. `netobserv-flowcollector-ready` (inform): FlowCollector `Ready`.

Compliant therefore means the stack actually works, not just that the objects
exist.

| Feature | Shows | Requirement |
|---|---|---|
| `PacketDrop` | packets dropped by the kernel / OVS, with the drop reason | privileged agent |
| `DNSTracking` | DNS queries, response codes and latency | — |
| `FlowRTT` | TCP round-trip time per flow | — |

**Verify:**

```
oc get lokistack loki -n netobserv
oc get flowcollector cluster -o jsonpath='{.status.conditions[?(@.type=="Ready")]}'
oc get pods -n netobserv; oc get pods -n netobserv-privileged
```

## GCS on GCP

Detected from the WIF audience in the CCO-managed
`openshift-ingress-operator/cloud-credentials` secret. Steps 2-4 then create
Secret `loki-gcs` (only `bucketname`) instead of the ObjectBucketClaim and
`loki-s3`, and the LokiStack uses `type: gcs`. The Loki Operator creates a
CredentialsRequest `netobserv/loki`; the CCO writes
`netobserv/loki-managed-credentials`, and Loki exchanges its ServiceAccount
token with GCP STS for a token impersonating
`openshift-loki@<project ID>.iam.gserviceaccount.com`. No keys are stored.

| Discovered on the cluster | Value |
|---|---|
| Bucket | `<GCP project ID>-<infrastructure name>-loki` (max 63 characters) |
| GCP service account | `openshift-loki@<GCP project ID>.iam.gserviceaccount.com` |
| Kubernetes ServiceAccounts | `netobserv/loki`, `netobserv/loki-ruler` |

**Per-cluster prerequisite (one time).** The CCO cannot create buckets, service
accounts or IAM bindings in Manual mode. Run with the cluster's kubeconfig and
gcloud logged in to the cluster's project:

```bash
PROJECT_ID=$(oc get infrastructure cluster -o jsonpath='{.status.platformStatus.gcp.projectID}')
REGION=$(oc get infrastructure cluster -o jsonpath='{.status.platformStatus.gcp.region}')
INFRA=$(oc get infrastructure cluster -o jsonpath='{.status.infrastructureName}')
AUDIENCE=$(oc get secret cloud-credentials -n openshift-ingress-operator \
  -o jsonpath='{.data.service_account\.json}' | base64 -d | jq -r .audience)
POOL=${AUDIENCE#//iam.googleapis.com/}; POOL=${POOL%/providers/*}
BUCKET=${PROJECT_ID}-${INFRA}-loki
GSA=openshift-loki@${PROJECT_ID}.iam.gserviceaccount.com

gcloud storage buckets create gs://${BUCKET} --project ${PROJECT_ID} \
  --location ${REGION} --uniform-bucket-level-access
gcloud iam service-accounts describe ${GSA} --project ${PROJECT_ID} >/dev/null 2>&1 || \
  gcloud iam service-accounts create openshift-loki --project ${PROJECT_ID}
gcloud storage buckets add-iam-policy-binding gs://${BUCKET} \
  --member "serviceAccount:${GSA}" --role roles/storage.objectAdmin
for KSA in loki loki-ruler; do
  gcloud iam service-accounts add-iam-policy-binding ${GSA} --project ${PROJECT_ID} \
    --role roles/iam.workloadIdentityUser \
    --member "principal://iam.googleapis.com/${POOL}/subject/system:serviceaccount:netobserv:${KSA}"
done
```

Until this is done every policy is Compliant and the LokiStack is Ready, but
the ingester cannot flush flows to the bucket (`Gaia id not found for email
openshift-loki@...` or a 403 in `oc logs -n netobserv loki-ingester-0`), and
flows older than the ingester's in-memory window are lost. Check the write path
without waiting for a regular flush:

```bash
oc exec -n netobserv loki-ingester-0 -- curl -sk -X POST https://localhost:3100/flush \
  --cert /var/run/tls/http/server/tls.crt --key /var/run/tls/http/server/tls.key
oc logs -n netobserv loki-ingester-0 --since=2m | grep -i 'failed to flush'   # empty = OK
gcloud storage ls gs://${BUCKET}/network/ | head
```

## Current setup (single node)

`1x.demo` is the smallest LokiStack: a single replica of every component with
replication factor 1. The FlowCollector therefore permanently reports the Loki
integration as `Degraded` ("ingester replicas (1) are less than or equal to the
replication factor"). Flow ingestion pauses whenever the ingester restarts.
This size is **not for production**.

## Required changes for 3 infra nodes

All changes are in `day2-netobserv-flowcollector.yaml`.

1. **LokiStack size:** change `size: 1x.demo` to **`1x.pico`**, the smallest
   size with replication factor 2 and replicated components, which needs at
   least 3 nodes. Use `1x.extra-small` for larger clusters or longer retention.
   Check that the default StorageClass can provide the larger PVCs.
2. **eBPF agent and flowlogs-pipeline on every node, including infra nodes.**
   In `Direct` mode both run as DaemonSets. If they don't tolerate the infra
   taint, flows from the infra nodes (the routers) are missing. Add:
   ```yaml
   spec:
     agent:
       ebpf:
         advanced:
           scheduling:
             tolerations:
               - operator: Exists
     processor:
       advanced:
         scheduling:
           tolerations:
             - operator: Exists
   ```
   Then check `oc get ds -n netobserv-privileged netobserv-ebpf-agent` and
   `oc get ds -n netobserv flowlogs-pipeline`: DESIRED should equal the node count.
3. **Optional: Loki on the infra nodes.** Add to the LokiStack, for each
   component (`compactor`, `distributor`, `gateway`, `indexGateway`,
   `ingester`, `querier`, `queryFrontend`, `ruler`):
   ```yaml
   spec:
     template:
       ingester:
         nodeSelector:
           node-role.kubernetes.io/infra: ""
         tolerations:
           - key: node-role.kubernetes.io/infra
             operator: Exists
   ```
4. **Optional:** with more nodes there are more flows. Raise `sampling` (for
   example to 100) if Loki or the agents use too much memory.
