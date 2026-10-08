# platform-ops policies

All objects live in the `rhacm-policies` namespace on the hub.

```
platform-ops/
├── managedclustersetbinding.yaml   # binds "global"; every Placement depends on it
├── baseline/                       # every OpenShift cluster, no cluster-specific parameters
│   ├── placement.yaml              # placement-openshift-day2 (vendor=OpenShift), binds all sections
│   ├── security/                   # etcd encryption, self-provisioner, kubeadmin, OAuth tokens, network policies
│   ├── compliance/                 # Compliance Operator + PCI-DSS v4.0 scan and results
│   ├── pipelines/                  # OpenShift Pipelines
│   ├── operators/                  # Gatekeeper, Descheduler
│   ├── infrastructure/             # infra MCP; ingress, monitoring, registry on infra nodes
│   └── cluster-settings/           # console banner, Remote Health Reporting opt-out
├── gcp-logging/                    # cloud=Google clusters
└── network-observability/          # opt-in: network-observability=enabled
```

- `baseline/` holds the policies applied to every OpenShift cluster. They take
  no cluster-specific parameters. Each section folder has its own PolicySet
  (`baseline-<section>`), and `baseline/placement.yaml` binds all of them to
  one Placement.
- Every other top-level folder holds one policy group that needs
  cluster-specific parameters or targets a subset of clusters. Each is
  self-contained: its policies, its own `policyset.yaml`, and its own
  `placement.yaml` (Placement + PlacementBinding).
  - `gcp-logging/`: log forwarding to Google Cloud Logging (`cloud=Google`).
  - `network-observability/`: Network Observability with a privileged eBPF
    agent (PacketDrop, DNSTracking, FlowRTT) and a `1x.demo` LokiStack on MCG
    object storage. Opt-in: label the cluster `network-observability=enabled`.

To add a baseline policy, put it in the matching section folder and list it in
that folder's `policyset.yaml`. For a new section, create the folder with a
`policyset.yaml` and add the PolicySet to the subjects in
`baseline/placement.yaml`. For a cluster-specific group, create a new top-level
folder following the `gcp-logging/` layout.

Apply recursively (Argo CD: Directory Recurse = true, or `oc apply -R -f platform-ops`).
