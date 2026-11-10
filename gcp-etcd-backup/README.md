# etcd backup to Google Cloud Storage (`openshift-gcp-etcd-backup`)

Backs up etcd every 6 hours to a Cloud Storage bucket. Authentication uses GCP
Workload Identity Federation (WIF): the Cloud Credential Operator (CCO) issues a
short-lived credential config, so no service account key is stored on the cluster.

**Opt-in:** targets clusters labelled `vendor=OpenShift`, `cloud=Google` **and**
`etcd-backup=gcs`. Add the label only after the requirements below are met.

| Policy | Mode | What it does |
|---|---|---|
| `day2-etcd-backup-gcs` | enforce + inform check | 1. Namespace `ocp-etcd-backup` (pod security `privileged`), ServiceAccount `etcd-backup` bound to the `privileged` SCC, CredentialsRequest `etcd-backup-gcs` (the CCO writes Secret `ocp-etcd-backup/etcd-backup-gcp-wif`). 2. CronJob `etcd-backup`, every 6 hours on one control-plane node. 3. Inform check: NonCompliant when no backup succeeded in the last 13 hours. |

## How a backup runs

1. **Init container** (`ubi9`, privileged, host network and PID, like
   `oc debug node`): `chroot /host /usr/local/bin/cluster-backup.sh
   /var/tmp/etcd-backup` writes `snapshot_<ts>.db` and
   `static_kuberesources_<ts>.tar.gz` on the node.
2. **Upload container** (`google-cloud-cli:slim`): `gcloud auth login
   --cred-file` with the CCO-issued `external_account` config, which exchanges
   the pod's ServiceAccount token (audience `openshift`) with GCP STS. Each file
   is then uploaded with one Cloud Storage JSON API request (`curl`, with
   `ifGenerationMatch=0` so nothing is overwritten) to
   `gs://<bucket>/<infrastructure name>/<UTC timestamp>/`, and the node
   directory is emptied. `gcloud storage cp` is not used for the upload because
   it also reads the destination (`storage.objects.get`), which the
   create-only role does not allow.
3. **Retention** is a lifecycle rule on the bucket. The job can only create
   objects: it cannot read, overwrite or delete existing backups.

| Discovered on the cluster | Value |
|---|---|
| Bucket | `<project ID>-<infrastructure name>-etcd-backup` |
| Object prefix | `<infrastructure name>` |
| GCP service account (default) | `openshift-etcd-backup@<project ID>.iam.gserviceaccount.com` |
| Kubernetes ServiceAccount | `ocp-etcd-backup/etcd-backup` |

With `ccoctl` the GCP service account gets a generated name. Put it in key
`<cluster>.etcd-backup` of the hub ConfigMap
`rhacm-policies/gcp-wif-service-accounts`, as for logging and Loki.

## Requirements and how to meet them

| # | Requirement | How |
|---|---|---|
| R1 | OpenShift on GCP installed with WIF (CCO in Manual mode) | Check: `oc get secret cloud-credentials -n openshift-ingress-operator -o jsonpath='{.data.service_account\.json}' \| base64 -d \| jq -r .type` prints `external_account` |
| R2 | A dedicated, private bucket with uniform access, no public access and a retention rule | Step 2 |
| R3 | A GCP service account allowed only to create objects in that bucket | Step 3 |
| R4 | The cluster's ServiceAccount `ocp-etcd-backup/etcd-backup` may impersonate it through the WIF pool | Step 3 |
| R5 | Egress from the control-plane nodes to `storage.googleapis.com`, `sts.googleapis.com`, `iamcredentials.googleapis.com`, and image pulls from `registry.access.redhat.com` and `gcr.io` | Firewall/proxy; for a disconnected cluster mirror both images and change them in the policy |
| R6 | The bucket is treated as highly sensitive | Step 4. `static_kuberesources_*.tar.gz` contains the API server encryption keys, so anyone who can read both files can read every Secret |
| R7 | The policy is deployed by GitOps | Step 5 |
| R8 | Restores are tested | Step 7 |

### Step 1: variables

Run with `oc` logged in to the cluster and `gcloud` logged in to its project.

```bash
PROJECT_ID=$(oc get infrastructure cluster -o jsonpath='{.status.platformStatus.gcp.projectID}')
REGION=$(oc get infrastructure cluster -o jsonpath='{.status.platformStatus.gcp.region}')
INFRA=$(oc get infrastructure cluster -o jsonpath='{.status.infrastructureName}')
AUDIENCE=$(oc get secret cloud-credentials -n openshift-ingress-operator \
  -o jsonpath='{.data.service_account\.json}' | base64 -d | jq -r .audience)
POOL_PATH=${AUDIENCE#//iam.googleapis.com/}; POOL_PATH=${POOL_PATH%/providers/*}
BUCKET=${PROJECT_ID}-${INFRA}-etcd-backup
GSA=openshift-etcd-backup@${PROJECT_ID}.iam.gserviceaccount.com
```

### Step 2: bucket (R2)

```bash
gcloud storage buckets create "gs://${BUCKET}" --project "$PROJECT_ID" \
  --location "$REGION" --uniform-bucket-level-access --public-access-prevention

cat > lifecycle.json <<'EOF'
{"rule": [{"action": {"type": "Delete"}, "condition": {"age": 14}}]}
EOF
gcloud storage buckets update "gs://${BUCKET}" --lifecycle-file=lifecycle.json
```

14 days at 4 backups per day keeps 56 backups. Optional: a dual-region or
multi-region location so a regional outage cannot take the backups with the
cluster, and `--retention-period` with a lock if backups must be immutable.

### Step 3: service account and WIF binding (R3, R4)

```bash
gcloud iam service-accounts create openshift-etcd-backup --project "$PROJECT_ID" \
  --display-name "OpenShift etcd backup (${INFRA})"
gcloud storage buckets add-iam-policy-binding "gs://${BUCKET}" \
  --member "serviceAccount:${GSA}" --role roles/storage.objectCreator
gcloud iam service-accounts add-iam-policy-binding "$GSA" --project "$PROJECT_ID" \
  --role roles/iam.workloadIdentityUser \
  --member "principal://iam.googleapis.com/${POOL_PATH}/subject/system:serviceaccount:ocp-etcd-backup:etcd-backup"
```

With `ccoctl` instead: after the policy has created CredentialsRequest
`etcd-backup-gcs`, follow [docs/gcp-wif-ccoctl.md](../docs/gcp-wif-ccoctl.md)
with that CredentialsRequest and key `<cluster>.etcd-backup`. `ccoctl` grants
`roles/storage.objectCreator` on the whole project; prefer the bucket binding
above.

### Step 4: protect the backups (R6)

- Grant read access (`roles/storage.objectViewer`) only to a small break-glass
  group, never to the cluster's service account.
- Optional: encrypt the bucket with a customer-managed key
  (`gcloud storage buckets update "gs://${BUCKET}" --default-encryption-key=<KMS key>`;
  the Cloud Storage service agent needs `roles/cloudkms.cryptoKeyEncrypterDecrypter`).
- Turn on Data Access audit logs for Cloud Storage to record every read.

### Step 5: deploy through GitOps (R7)

The ApplicationSet `baseline-policies` only syncs `baseline/`. Add an
ApplicationSet (or a source) for `gcp-etcd-backup/`, then opt the cluster in:

```bash
oc label managedcluster local-cluster etcd-backup=gcs
```

### Step 6: verify

```bash
oc get secret etcd-backup-gcp-wif -n ocp-etcd-backup     # written by the CCO
oc create job --from=cronjob/etcd-backup etcd-backup-manual -n ocp-etcd-backup
oc logs -n ocp-etcd-backup job/etcd-backup-manual -c snapshot
oc logs -n ocp-etcd-backup job/etcd-backup-manual -c upload
gcloud storage ls "gs://${BUCKET}/${INFRA}/"
```

`day2-etcd-backup-gcs` becomes Compliant once a run has succeeded.

### Step 7: test a restore (R8)

Download one backup folder to a control-plane node and follow
**Restoring to a previous cluster state** in the OpenShift backup and restore
documentation (`/usr/local/bin/cluster-restore.sh <dir>`). Practice it on a
non-production cluster; a restore is disruptive.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Pod stuck `CreateContainerConfigError`, Secret `etcd-backup-gcp-wif` not found | The CCO has not processed the CredentialsRequest | `oc get credentialsrequest etcd-backup-gcs -n openshift-cloud-credential-operator -o yaml` |
| `upload`: `Gaia id not found` or `PERMISSION_DENIED` on `generateAccessToken` | Service account missing or no workloadIdentityUser binding | Step 3 |
| `upload`: `403` on the bucket | Missing `roles/storage.objectCreator` | Step 3 |
| `(gcloud.storage.cp) HTTPError 403 … does not have storage.objects.get access` | An older version of the policy used `gcloud storage cp`, which reads the destination | Use the current policy (JSON API upload); do not grant read access to the backup service account |
| `snapshot` fails | etcd unhealthy or not enough disk on the node | `oc get etcd -o=jsonpath='{range .items[0].status.conditions[*]}{.type}={.status}{"\n"}{end}'` and node disk |
| Policy NonCompliant, annotation `no-successful-etcd-backup-in-the-last-13h` | Two runs in a row failed, or no run yet | `oc get jobs -n ocp-etcd-backup` and the logs above |
