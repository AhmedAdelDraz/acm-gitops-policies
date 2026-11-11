# GCP prerequisites with `ccoctl` (logging and network observability)

On OpenShift on GCP with Workload Identity Federation (WIF) the Cloud Credential
Operator runs in Manual mode: it writes short-lived credential Secrets for
CredentialsRequests, but it cannot create GCP service accounts, IAM bindings or
buckets. These are created once per cluster, here with `ccoctl`.

| Group | CredentialsRequest | Kubernetes ServiceAccounts | GCP role | Extra GCP object |
|---|---|---|---|---|
| GCP logging | `openshift-logging-gcp-wif` (created by `day2-logging-gcp-forwarder`) | `openshift-logging/logcollector` | `roles/logging.logWriter` | — |
| Network observability (Loki) | written by hand below (the Loki Operator creates its own only after the LokiStack exists) | `netobserv/loki`, `netobserv/loki-ruler` | `roles/storage.objectAdmin` | GCS bucket `<project ID>-<infrastructure name>-loki` |

## How the policies find the service accounts

`ccoctl` names each GCP service account
`<--name, 12 chars>-<CredentialsRequest name, 11 chars>-<5 random chars>`, so
the email cannot be predicted. Both policies read it from the hub ConfigMap
`rhacm-policies/gcp-wif-service-accounts`:

| Key | Used by |
|---|---|
| `<managed cluster name>.logging` | `serviceAccountEmail` of the logging CredentialsRequest |
| `<managed cluster name>.loki` | `SERVICE_ACCOUNT_EMAIL` of the Loki Operator, which puts it in its own CredentialsRequest |

Without the key the policies fall back to the fixed names used with plain
`gcloud`: `openshift-logging-writer@<project ID>` and `openshift-loki@<project ID>`.

Do **not** apply the Secret manifests `ccoctl` writes. The CCO creates the Secrets
from the CredentialsRequests, and the logging collector reads its token from
`/var/run/ocp-collector/serviceaccount/token`, not from the path `ccoctl` puts in
its manifests.

## Steps

Run on a **Linux** host (`ccoctl` is shipped for Linux only), logged in to the
cluster with `oc` and with `jq` installed. The GCP identity needs
`roles/iam.serviceAccountAdmin`, `roles/resourcemanager.projectIamAdmin` and,
for the bucket, `roles/storage.admin` on the project.

### 1. Values

```bash
PROJECT_ID=$(oc get infrastructure cluster -o jsonpath='{.status.platformStatus.gcp.projectID}')
REGION=$(oc get infrastructure cluster -o jsonpath='{.status.platformStatus.gcp.region}')
INFRA=$(oc get infrastructure cluster -o jsonpath='{.status.infrastructureName}')
AUDIENCE=$(oc get secret cloud-credentials -n openshift-ingress-operator \
  -o jsonpath='{.data.service_account\.json}' | base64 -d | jq -r .audience)
POOL=$(echo "$AUDIENCE" | cut -d/ -f9)        # workloadIdentityPools/<pool>
PROVIDER=$(echo "$AUDIENCE" | cut -d/ -f11)   # providers/<provider>
CLUSTER=local-cluster                          # ManagedCluster name on the hub
echo "$PROJECT_ID $REGION $INFRA $POOL $PROVIDER"
```

For the current hub this prints
`openenv-fs9jx us-central1 ocp4-l49rg ocp4-lab ocp4-lab`.

### 2. Get `ccoctl` from the cluster's release image

```bash
RELEASE_IMAGE=$(oc get clusterversion version -o jsonpath='{.status.desired.image}')
oc get secret pull-secret -n openshift-config \
  -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d > pull-secret.json
oc adm release extract --registry-config pull-secret.json \
  --command=ccoctl --to=./bin "$RELEASE_IMAGE"
chmod +x bin/ccoctl
rm pull-secret.json
```

### 3. Log in to GCP

`ccoctl` uses Application Default Credentials:

```bash
gcloud auth application-default login
gcloud config set project "$PROJECT_ID"
```

### 4. Collect the CredentialsRequests

```bash
mkdir -p credreqs

# Logging: the policy has already created it on the cluster.
oc get credentialsrequest openshift-logging-gcp-wif \
  -n openshift-cloud-credential-operator -o yaml > credreqs/logging.yaml

# Loki: only read by ccoctl, never applied to the cluster.
cat > credreqs/loki.yaml <<'EOF'
apiVersion: cloudcredential.openshift.io/v1
kind: CredentialsRequest
metadata:
  name: loki-netobserv
  namespace: openshift-cloud-credential-operator
spec:
  serviceAccountNames:
    - loki
    - loki-ruler
  secretRef:
    name: loki-managed-credentials
    namespace: netobserv
  providerSpec:
    apiVersion: cloudcredential.openshift.io/v1
    kind: GCPProviderSpec
    predefinedRoles:
      - roles/storage.objectAdmin
    skipServiceCheck: true
EOF
```

`ccoctl` grants `predefinedRoles` on the **project**, so the Loki service account
can read and write every bucket in it. For a bucket-only grant, remove
`predefinedRoles` here and run the bucket binding in step 6.

### 5. Create the service accounts

Use `create-service-accounts`, **not** `create-all`: the pool, provider and OIDC
issuer were created at install time and must stay as they are.

```bash
./bin/ccoctl gcp create-service-accounts \
  --name="$INFRA" \
  --project="$PROJECT_ID" \
  --workload-identity-pool="$POOL" \
  --workload-identity-provider="$PROVIDER" \
  --credentials-requests-dir=credreqs \
  --output-dir=ccoctl-out
```

For each CredentialsRequest this creates the GCP service account, grants its
`predefinedRoles` on the project, and grants `roles/iam.workloadIdentityUser`
to each Kubernetes ServiceAccount through the pool.

Read the generated emails:

```bash
gsa() {
  grep 'service_account.json' "$1" | awk '{print $2}' | base64 -d \
    | jq -r .service_account_impersonation_url \
    | sed -e 's#.*/serviceAccounts/##' -e 's#:generateAccessToken##'
}
LOGGING_GSA=$(gsa ccoctl-out/manifests/openshift-logging-gcp-wif-secret-credentials.yaml)
LOKI_GSA=$(gsa ccoctl-out/manifests/netobserv-loki-managed-credentials-credentials.yaml)
echo "$LOGGING_GSA"; echo "$LOKI_GSA"
```

### 6. Create the Loki bucket

`ccoctl` does not create buckets. The name must match what the policy derives:

```bash
BUCKET=${PROJECT_ID}-${INFRA}-loki
gcloud storage buckets create "gs://${BUCKET}" --project "$PROJECT_ID" \
  --location "$REGION" --uniform-bucket-level-access
# Only if predefinedRoles was removed in step 4:
# gcloud storage buckets add-iam-policy-binding "gs://${BUCKET}" \
#   --member "serviceAccount:${LOKI_GSA}" --role roles/storage.objectAdmin
```

### 7. Hand the emails to the policies (on the hub)

```bash
oc create configmap gcp-wif-service-accounts -n rhacm-policies \
  --from-literal="${CLUSTER}.logging=${LOGGING_GSA}" \
  --from-literal="${CLUSTER}.loki=${LOKI_GSA}" \
  --dry-run=client -o yaml | oc apply -f -
```

The ConfigMap is not in Git, so Argo CD leaves it alone. ACM re-renders both
policies when it changes; the CCO then rewrites
`openshift-logging/gcp-wif-secret` with the new service account.

### 8. Restart the log collector

The collector caches a failed token exchange ("Subsequent calls with this
credential will also fail"), so restart it once the Secret has the new email:

```bash
oc get secret gcp-wif-secret -n openshift-logging \
  -o jsonpath='{.data.service_account\.json}' | base64 -d | jq -r .service_account_impersonation_url
oc delete pod -n openshift-logging -l app.kubernetes.io/component=collector
```

### 9. Turn on network observability

```bash
oc label managedcluster "$CLUSTER" network-observability=enabled
```

## Verify

```bash
# logging: no token errors, logs in Logs Explorer under the cluster's log ID
oc logs -n openshift-logging ds/gcp-logging -c collector --since=10m | grep -c 'Failed to get GCP access token'

# network observability
oc get lokistack loki -n netobserv
oc get credentialsrequest -A | grep -i loki
oc get secret loki-managed-credentials -n netobserv
oc get flowcollector cluster -o jsonpath='{.status.conditions[?(@.type=="Ready")]}'
```

## Removing

Do not use `ccoctl gcp delete` for this: it is meant for cluster teardown and
also removes the workload identity pool and OIDC bucket named after `--name`,
which the cluster itself still needs. Delete only the two service accounts (their
IAM bindings go with them) and remove the ConfigMap keys:

```bash
gcloud iam service-accounts delete "$LOGGING_GSA" --project "$PROJECT_ID"
gcloud iam service-accounts delete "$LOKI_GSA" --project "$PROJECT_ID"
gcloud projects get-iam-policy "$PROJECT_ID" --format=json | grep -c deleted:serviceAccount   # stale project bindings to clean up
```
