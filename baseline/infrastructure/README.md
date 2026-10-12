# Infrastructure (`baseline-infrastructure`)

Places the platform's infrastructure workloads (routers, monitoring, image
registry) on dedicated infra nodes. Every policy first looks up nodes labelled
`node-role.kubernetes.io/infra` on the managed cluster and **only moves
workloads when such nodes exist**. Clusters without infra nodes, such as the
current single-node hub, keep their placement, so the group is safe to enforce
everywhere.

| Policy | Mode | What it does |
|---|---|---|
| `day2-infra-machineconfigpool` | enforce | Creates the `infra` MachineConfigPool. It selects nodes labelled `node-role.kubernetes.io/infra` and renders worker MachineConfigs plus any with role `infra`. |
| `day2-ingress-infra` | enforce | With infra nodes: default IngressController on infra nodes, one replica per infra node (max 3), tolerating the infra taint. The node selector is **replaced**, not merged, and every other field (default certificate, endpoint strategy, TLS) is preserved. |
| `day2-monitoring-infra` | enforce | `cluster-monitoring-config`: user workload monitoring on, Prometheus retention 15d. With infra nodes: all platform monitoring components on infra nodes. With a default StorageClass: Prometheus gets a 100Gi PVC and Alertmanager 5Gi. |
| `day2-image-registry-infra` | enforce | With infra nodes: registry on infra nodes, tolerating the infra taint. Replicas are raised to one per infra node (max 3) **only on object storage**. Storage and `managementState` are left as configured. `recordDiff` is off because the spec contains the registry `httpSecret`. |

## Current setup (single node)

There are no infra nodes, so:

- the `infra` pool exists with 0 machines;
- the router and registry keep their placement (no change);
- monitoring gets retention, user workload monitoring and PVCs on the default
  StorageClass (`ocs-external-storagecluster-ceph-rbd`).

## Required changes for 3 infra nodes

**Prerequisites (once per cluster):** label and taint the nodes, and repoint
`*.apps`. See [the top-level README](../../README.md#moving-from-single-node-to-3-infra-nodes).

**Happens automatically** on the next evaluation once the nodes are labelled:

- the 3 nodes join the `infra` MachineConfigPool (each one reboots once while
  it moves into the pool);
- the default IngressController runs **3 replicas** on the infra nodes;
- every monitoring component moves to the infra nodes. On a multi-node cluster
  Prometheus and Alertmanager run 2 replicas each, so storage doubles
  (2 × 100Gi + 2 × 5Gi);
- the registry moves to the infra nodes.

**Manual change: image registry high availability.** On this cluster the
registry uses a ReadWriteOnce Ceph RBD PVC (`pvc-image-registry`), which only
one pod can mount, so the policy keeps it at **1 replica**. For 3 replicas,
move the registry to object storage. The policy then raises it to 3
automatically. With ODF, use the Multicloud Object Gateway:

1. Create an ObjectBucketClaim (`storageClassName: openshift-storage.noobaa.io`)
   in `openshift-image-registry`.
2. Set `spec.storage.s3` in `configs.imageregistry.operator.openshift.io/cluster`
   to the bucket from the generated ConfigMap, with the credentials from the
   generated Secret in `openshift-image-registry/image-registry-private-configuration-user`.
3. Remove `spec.storage.pvc`.

A ReadWriteMany CephFS PVC also supports several replicas, but the policy does
not scale PVC-backed registries. Set `replicas` yourself in that case.

**Sizing.** Infra nodes then carry the routers, the whole monitoring stack and
the registry. Size them using Red Hat's infrastructure node sizing guidance for
the expected number of worker nodes and pods.

**Verify:**

```
oc get mcp infra                                  # MACHINECOUNT 3, UPDATED True
oc get pods -n openshift-ingress -o wide          # 3 router pods on infra nodes
oc get pods -n openshift-monitoring -o wide       # all on infra nodes
oc get pods -n openshift-image-registry -o wide
```
