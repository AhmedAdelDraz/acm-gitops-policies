# platform-ops policies

All objects live in the `rhacm-policies` namespace on the hub.

- `managedclustersetbinding.yaml` binds the `global` ManagedClusterSet to
  `rhacm-policies`. Every Placement below depends on it.
- `baseline/` holds the policies applied to every OpenShift cluster
  (`vendor=OpenShift`). They take no cluster-specific parameters, so they share
  one PolicySet (`openshift-day2-operations`) and one Placement.
- Every other folder holds one policy group that needs cluster-specific
  parameters or targets a subset of clusters. Each folder is self-contained:
  its policies, its own `policyset.yaml`, and its own `placement.yaml`
  (Placement + PlacementBinding).
  - `gcp-logging/`: log forwarding to Google Cloud Logging (`cloud=Google`).

To add a baseline policy, drop it into `baseline/` and list it in
`baseline/policyset.yaml`. To add a cluster-specific one, create a new folder
following the `gcp-logging/` layout.

Apply recursively (Argo CD: Directory Recurse = true, or `oc apply -R -f platform-ops`).
