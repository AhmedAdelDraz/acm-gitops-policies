# Pipelines (`baseline-pipelines`)

Installs OpenShift Pipelines (Tekton) on every cluster.

| Policy | Mode | What it does |
|---|---|---|
| `day2-openshift-pipelines` | enforce | Subscribes to `openshift-pipelines-operator-rh` from `redhat-operators`, channel `latest`, in `openshift-operators` (cluster-wide, uses the existing global OperatorGroup), automatic upgrades. |

The operator creates its default `TektonConfig` (profile `all`) itself, which
deploys Pipelines, Triggers, Chains, Pipelines as Code and the console
integration into `openshift-pipelines`.

- **Pin a version:** replace `channel: latest` with a release channel such as
  `pipelines-1.24` to control upgrades.
- **Air-gapped clusters:** `source` must name the mirrored CatalogSource.

**Verify:**

```
oc get csv -n openshift-operators | grep pipelines
oc get tektonconfig config -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}'
```

## Required changes for 3 infra nodes

**None.** The Pipelines controllers and every PipelineRun pod belong on worker
nodes. Red Hat does not count them as infrastructure workloads, and running
them on infra nodes would make those nodes subscription-billable. With the
infra taint in place, they stay on the workers automatically.
