# Security (`baseline-security`)

Platform hardening applied to every OpenShift cluster.

| Policy | Mode | What it does |
|---|---|---|
| `day2-etcd-encryption` | enforce | `APIServer/cluster` `spec.encryption.type: aescbc`: Secrets, ConfigMaps, Routes, OAuth tokens and other sensitive resources are encrypted at rest in etcd. |
| `day2-disable-self-provisioner` | enforce | Removes the rules of the `self-provisioner` ClusterRole and the subjects of the `self-provisioners` binding, with `autoupdate: "false"` so the API server does not restore them. Regular users can no longer create projects. |
| `day2-htpasswd-identity-provider` | enforce | HTPasswd identity provider `htpasswd` (users file `openshift-config/htpasswd-users`, copied from the hub Secret `rhacm-policies/htpasswd-users`) and `ClusterRoleBinding/ocpadmin-cluster-admin`. An inform check reports when the `authentication` operator has rolled it out. |
| `day2-kubeadmin-removed` | enforce (guarded) | Depends on `day2-htpasswd-identity-provider`. Deletes `kube-system/kubeadmin` **only if** an identity provider is configured **and** a non-system User or Group is bound to `cluster-admin`. Otherwise the inform check `kubeadmin-removal-prerequisites` is NonCompliant and names the missing prerequisite (`missing-identity-provider-…` or `missing-cluster-admin-user-…`), and kubeadmin is kept. |
| `day2-oauth-token-duration` | enforce | `OAuth/cluster` `accessTokenMaxAgeSeconds: 3600`: OAuth access tokens expire after 1 hour (default 24h). |
| `day2-default-network-policies` | enforce | Namespace isolation for every application namespace (see below). |

## Operational impact

- **etcd encryption:** the first rollout restarts the API servers. On a single
  node this briefly makes `openshift-apiserver`, `authentication` and
  `image-registry` unavailable (about 5 minutes on the hub). Progress:
  `oc get openshiftapiserver cluster -o jsonpath='{.status.conditions[?(@.type=="Encrypted")]}'`.
- **kubeadmin removal cannot be undone.** Make sure the identity provider
  login works for at least one cluster-admin before placing this policy.
- **1-hour tokens:** `oc login` sessions, and tools that store a user token,
  must log in again every hour. Use ServiceAccount tokens for automation.
  Existing tokens keep their original lifetime.

## HTPasswd users (hub Secret)

The users file is kept out of Git. Create it once on the hub, in the policy
namespace, with bcrypt hashes; the policy copies it (encrypted by ACM in
transit) to every cluster it is placed on:

```bash
htpasswd -cbB users.htpasswd ocpadmin '<password>'      # -c only for a new file
oc create secret generic htpasswd-users -n rhacm-policies \
  --from-file=htpasswd=users.htpasswd --dry-run=client -o yaml | oc apply -f -
rm users.htpasswd
```

To add or change a user, rebuild the file with all users (or extract it with
`oc extract secret/htpasswd-users -n rhacm-policies --to=-`), update the Secret,
and the policy pushes it on its next evaluation. Only `ocpadmin` is bound to
`cluster-admin`; other users get the default role.

Without the hub Secret the policy is NonCompliant and kubeadmin is kept.

**Before kubeadmin is removed,** log in once as ocpadmin and check the role:

```bash
oc login -u ocpadmin <api-url>
oc auth can-i '*' '*' --all-namespaces    # yes
```

## Default network policies

In every namespace that is not excluded, three NetworkPolicies are created:

| NetworkPolicy | Allows ingress from |
|---|---|
| `allow-same-namespace` | pods in the same namespace |
| `allow-from-openshift-ingress` | the ingress routers (namespaces labelled `policy-group.network.openshift.io/ingress`), so Routes keep working |
| `allow-from-openshift-monitoring` | cluster and user workload monitoring (`network.openshift.io/policy-group: monitoring`), so metrics scraping keeps working |

- Egress is not restricted.
- NetworkPolicies are additive, so policies a namespace already has keep
  allowing what they allowed.
- **Excluded** (platform components talk across namespaces): `default`,
  `kube*`, `openshift*`, `open-cluster-management*`, `multicluster-engine`,
  `hive`, `hypershift`, `assisted-installer`, `cert-manager*`, `netobserv*`.
- **Opt-out:** `oc label ns <namespace> network-policy.day2.openshift.io/default=disabled`.
- New namespaces are covered on the next policy evaluation.

Two applications that must talk across namespaces need an extra NetworkPolicy
in the receiving namespace, for example allowing `namespaceSelector` with
`kubernetes.io/metadata.name: <client-namespace>`.

## Required changes for 3 infra nodes

**None.** None of these policies depend on node topology:

- the routers and monitoring are matched by namespace labels, not by node, so
  the network policies keep working when they move to infra nodes;
- the etcd encryption rollout is smoother on a 3-control-plane cluster, where
  the API servers restart one at a time without downtime.
