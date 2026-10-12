# Cluster settings (`baseline-cluster-settings`)

Cluster-wide settings and checks for every cluster.

| Policy | Mode | What it does |
|---|---|---|
| `day2-console-banner` | enforce | `ConsoleNotification` `restricted-environment-banner`: a black banner at the top of the web console reading "Restricted Environment - &lt;cluster name&gt;". The name comes from the managed cluster's `name` ClusterClaim. |
| `day2-disable-remote-health-reporting` | enforce | Opts the cluster out of Remote Health Reporting (Telemetry and Insights) the way Red Hat documents it: removes the `cloud.openshift.com` entry from `openshift-config/pull-secret`. Every other registry credential is kept as it is. |
| `day2-update-channel` | inform | Compliant when the update channel is `stable-*` or `eus-*`. Violation on `fast-*`, `candidate-*` or no channel; the violation shows the recommended `stable-<running minor>`. The channel is never changed. |

## Notes

- **Telemetry opt-out:** without Telemetry, Red Hat no longer receives cluster
  health data, and Insights recommendations and proactive support cases stop.
  Undo by
  adding the `cloud.openshift.com` credential back from
  console.redhat.com/openshift/downloads and removing the policy.
- **Fixing an update channel violation:**
  ```
  oc adm upgrade channel stable-4.22     # or eus-4.22 on an even minor release
  ```
  EUS channels exist for even minor versions only and are the choice for
  clusters that upgrade EUS-to-EUS.

## Required changes for 3 infra nodes

**None.** These settings are cluster-scoped objects with no workloads.
