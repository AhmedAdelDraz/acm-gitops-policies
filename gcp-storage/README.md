# GCP storage (`openshift-gcp-storage`)

Adds **ReadWriteMany** storage on GCP through the **GCP Filestore CSI driver**,
authenticating with **GCP Workload Identity Federation** (WIF), so no service
account keys are stored on the cluster.

ReadWriteOnce block storage needs no policy: OpenShift on GCP ships the
Persistent Disk CSI driver (`pd.csi.storage.gke.io`) with the `standard-csi`
(default) and `ssd-csi` StorageClasses.

**Targets** clusters labelled `vendor=OpenShift` and `cloud=Google`. Supported
only on OpenShift on GCP **installed with WIF** (Cloud Credential Operator in
Manual mode). Do not place it on a key-based GCP cluster.

| Policy | Mode | What it does |
|---|---|---|
| `day2-gcp-filestore-operator` | enforce | GCP Filestore CSI Driver Operator in `openshift-cluster-csi-drivers` (channel `stable`), with the WIF pool, provider, project number and GCP service account passed to it as environment variables. |
| `day2-gcp-filestore-driver` | enforce | Depends on the policy above. `ClusterCSIDriver/filestore.csi.storage.gke.io` (deploys the driver) and the `filestore-csi` StorageClass on the cluster VPC. An inform check reports when the WIF credentials secret exists and every driver controller replica is ready. |

## How authentication works

The operator puts the WIF values into the CredentialsRequest it creates, and
the CCO turns it into `openshift-cluster-csi-drivers/gcp-filestore-cloud-credentials`.
The driver's ServiceAccount token is exchanged with GCP STS for a token
impersonating `openshift-filestore-csi@<project ID>.iam.gserviceaccount.com`.
Everything cluster specific is discovered on the managed cluster:

- **Project number, pool, provider:** split out of the WIF audience in the
  CCO-managed `openshift-ingress-operator/cloud-credentials` secret;
- **GCP project ID** and **VPC** (`<infrastructure name>-network`, or
  `network-name` from `openshift-config/cloud-provider-config` on an existing
  VPC): from the cluster.

## Per-cluster prerequisite (one time)

In Manual mode the CCO cannot create GCP service accounts or IAM bindings.
Create them with `gcloud`; the exact commands are in the header of
`day2-gcp-filestore-driver.yaml`. In short:

1. Enable the Filestore API (`file.googleapis.com`).
2. Create `openshift-filestore-csi` in the cluster's project (once per project).
3. Grant it `roles/file.editor` and `roles/resourcemanager.tagUser`.
4. Grant `roles/iam.workloadIdentityUser` on it to the
   `gcp-filestore-csi-driver-operator` and `gcp-filestore-csi-driver-controller-sa`
   ServiceAccounts in `openshift-cluster-csi-drivers`.

Until this is done the policies apply cleanly, but the driver controller
crash-loops (`Gaia id not found for email openshift-filestore-csi@...` in the
`csi-driver` container log) and `day2-gcp-filestore-driver` stays NonCompliant.

**Verify:**

```
oc get clustercsidriver filestore.csi.storage.gke.io -o jsonpath='{.status.conditions}'
oc get pods -n openshift-cluster-csi-drivers | grep filestore
oc get sc filestore-csi
```

Test claim (creates a billed Filestore instance, about 5 minutes):

```
oc new-project filestore-test
oc create -f - <<'YAML'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rwx-test
spec:
  accessModes: ["ReadWriteMany"]
  storageClassName: filestore-csi
  resources:
    requests:
      storage: 1Ti
YAML
oc run rwx-test --image=registry.access.redhat.com/ubi9/ubi-minimal --restart=Never \
  --overrides='{"spec":{"volumes":[{"name":"d","persistentVolumeClaim":{"claimName":"rwx-test"}}],"containers":[{"name":"c","image":"registry.access.redhat.com/ubi9/ubi-minimal","command":["sh","-c","df -h /d && sleep 3600"],"volumeMounts":[{"name":"d","mountPath":"/d"}]}]}}'
oc delete project filestore-test   # deletes the Filestore instance
```

## Cost

`filestore-csi` uses the default Filestore tier (`standard`, Basic HDD): every
claim creates its own Filestore instance of at least **1 TiB**, billed while it
exists. It is therefore not the default StorageClass; use it only for claims that
need ReadWriteMany.

## Current setup

Deployed on the GCP SNO lab (`local-cluster`, OpenShift 4.22, WIF):
operator installed, WIF credentials issued by the CCO, driver and
`filestore-csi` StorageClass created, and the GCP STS token exchange accepted
the cluster's token. Volume provisioning waits for the GCP prerequisite above.

## Required changes for 3 infra nodes

None. The driver controller runs on control-plane nodes and the node DaemonSet
runs on every node.
