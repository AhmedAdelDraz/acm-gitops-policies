# acm-gitops-policies

Day-2 configuration for OpenShift clusters managed by Red Hat Advanced Cluster
Management (ACM), delivered as ACM Policies through GitOps.

All objects live in the `rhacm-policies` namespace on the hub. They are tested
on the current setup: a single-node OpenShift (SNO) 4.22 hub with ACM 2.17,
ODF external storage (Ceph RBD + Multicloud Object Gateway) and no infra nodes.
Every README explains what changes on a cluster with **3 dedicated infra nodes**.

## Layout

```
acm-gitops-policies/   (repository root)
├── managedclustersetbinding.yaml   # binds the "global" ClusterSet; every Placement depends on it
├── baseline/                       # every OpenShift cluster, no cluster-specific parameters
│   ├── placement.yaml              # placement-openshift-day2 (vendor=OpenShift), binds all sections
│   ├── security/                   # etcd encryption, self-provisioner, kubeadmin, OAuth tokens, network policies
│   ├── compliance/                 # Compliance Operator + PCI-DSS v4.0 scan and results
│   ├── pipelines/                  # OpenShift Pipelines
│   ├── operators/                  # Gatekeeper, Descheduler
│   ├── infrastructure/             # infra MCP; ingress, monitoring, registry on infra nodes
│   └── cluster-settings/           # console banner, telemetry opt-out, update channel check
├── gcp-logging/                    # cluster-specific: cloud=Google
└── network-observability/          # cluster-specific: opt-in label network-observability=enabled
```

| Group | PolicySet | Targets | README |
|---|---|---|---|
| Security | `baseline-security` | all OpenShift clusters | [baseline/security](baseline/security/README.md) |
| Compliance | `baseline-compliance` | all OpenShift clusters | [baseline/compliance](baseline/compliance/README.md) |
| Pipelines | `baseline-pipelines` | all OpenShift clusters | [baseline/pipelines](baseline/pipelines/README.md) |
| Operators | `baseline-operators` | all OpenShift clusters | [baseline/operators](baseline/operators/README.md) |
| Infrastructure | `baseline-infrastructure` | all OpenShift clusters | [baseline/infrastructure](baseline/infrastructure/README.md) |
| Cluster settings | `baseline-cluster-settings` | all OpenShift clusters | [baseline/cluster-settings](baseline/cluster-settings/README.md) |
| GCP logging | `openshift-gcp-logging` | `vendor=OpenShift`, `cloud=Google` | [gcp-logging](gcp-logging/README.md) |
| Network Observability | `openshift-network-observability` | `vendor=OpenShift`, `network-observability=enabled` | [network-observability](network-observability/README.md) |

## Conventions

- **Baseline vs. cluster-specific.** `baseline/` holds policies that need no
  per-cluster values; each section folder has its own PolicySet
  (`baseline-<section>`) and `baseline/placement.yaml` binds them all to one
  Placement. Every other top-level folder is a self-contained group with its
  own policies, `policyset.yaml` and `placement.yaml`.
- **Values are discovered, not hard-coded.** Templates use `lookup`,
  `fromSecret`, `fromClusterClaim` etc. on the managed cluster (node roles,
  default StorageClass, cluster name, GCP project), so one policy serves every
  cluster it is placed on.
- **Ordering.** `extraDependencies` (inside a policy) and `dependencies`
  (between policies) make templates wait for their prerequisites, e.g. an
  operator must be installed before its custom resource is created.
- **Remediation.** Everything is `enforce`, except checks that only report:
  `day2-compliance-pci-dss-results`, `day2-update-channel`, the inform guard in
  `day2-kubeadmin-removed` and the `*-ready` templates.
- **Operator channels are pinned** in OperatorPolicy. When two catalogs ship a
  package with the same name (e.g. `loki-operator` in redhat- and
  community-operators), an unpinned channel makes the install fail.

## Deploying

The hub needs the `global` ManagedClusterSet bound to `rhacm-policies`
(`managedclustersetbinding.yaml`); without it every Placement selects zero
clusters ("No placement found").

- **Argo CD:** an Application on path `.` (repository root) with
  **Directory Recurse = true** (`spec.source.directory.recurse: true`). Without
  recursion only the top-level file is synced and, with prune enabled, every
  policy in a subfolder is deleted.
- **Manually:** from the repository root, `oc apply -R -f .`

Check results:

```
oc get policyset -n rhacm-policies            # one line per group
oc get policy -n <managed-cluster-namespace>  # e.g. local-cluster, with compliance state
```

Removing a policy does not undo what it changed (encryption, pull secret,
operators). Remove the PlacementBinding first, then the policies.

## Moving from single node to 3 infra nodes

The policies detect infra nodes on each cluster and adapt automatically where
they can. What has to be done by hand, once per cluster, before or after the
policies are applied:

1. **Label the infra nodes** (on IPI clusters, set this in the infra
   MachineSet template instead):
   ```
   oc label node <infra-node> node-role.kubernetes.io/infra=""
   ```
2. **Taint them** so only infrastructure workloads run there (this also keeps
   them out of OpenShift subscription counts):
   ```
   oc adm taint node <infra-node> node-role.kubernetes.io/infra=reserved:NoSchedule
   ```
   Every infra-placed workload in these policies tolerates the
   `node-role.kubernetes.io/infra` key with any value or effect.
3. **Point `*.apps` at the infra nodes.** Once the routers move there, the
   external load balancer or DNS for the application wildcard must target the
   infra nodes instead of the old router nodes, or routes stop resolving.
4. Apply the group-specific changes below.

| Group | Automatic on 3 infra nodes | Manual change |
|---|---|---|
| Infrastructure | infra MCP gets the nodes; 3 router replicas on infra; monitoring on infra; registry on infra | registry stays at 1 replica on a ReadWriteOnce PVC: move it to object storage for 3 replicas |
| Network Observability | — | LokiStack `1x.demo` → `1x.pico` or larger; tolerations for the eBPF agent and flowlogs-pipeline; optionally Loki on infra |
| GCP logging | — | collector must tolerate the infra taint |
| Compliance | node scans keep working (scanner tolerates all taints) | optional: add the `infra` role to the ScanSetting |
| Operators | — | Descheduler: consider `Automatic` mode and more profiles |
| Security, Pipelines, Cluster settings | no change | none |

The single-node limit of 250 pods per node, which currently blocks the PCI-DSS
scan on the hub, does not apply to a multi-node cluster in the same way.
