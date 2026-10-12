# Operators (`baseline-operators`)

Day-2 platform operators installed on every cluster.

| Policy | Mode | What it does |
|---|---|---|
| `day2-gatekeeper-operator` | enforce | Gatekeeper Operator (`gatekeeper-operator-product`, `redhat-operators`) in `openshift-operators`, then a `Gatekeeper` instance: audit 1 replica, validating webhook on (2 replicas), **mutating webhook off**. |
| `day2-descheduler-operator` | enforce | Kube Descheduler Operator in `openshift-kube-descheduler-operator`, then `KubeDescheduler/cluster`: mode **`Predictive`** (evaluates and reports evictions, evicts nothing), profile `AffinityAndTaints`, every 3600s. |

## Gatekeeper

The instance on its own enforces nothing. It is the engine for
`ConstraintTemplate` + constraint objects, which can be delivered as further
policies, for example required labels, allowed registries, or disallowing
`latest` tags. Start constraints with `enforcementAction: dryrun` or `warn` and
check audit results before switching to `deny`:

```
oc get constraints
oc get <kind> <name> -o jsonpath='{.status.violations}'
```

## Descheduler

`Predictive` mode only reports what it would evict. Check the
`descheduler_*` metrics or the operator logs before switching to
`Automatic`.

## Required changes for 3 infra nodes

- **Gatekeeper:** no change is required. Its pods belong on worker nodes, not
  infra nodes. For webhook availability across node failures, consider
  `webhook.replicas: 3`.
- **Descheduler:** on a single node there is nowhere to move pods, so the
  descheduler has no effect on the current hub. On a multi-node cluster it
  becomes useful:
  - after reviewing the Predictive results, set `mode: Automatic`;
  - consider adding `LifecycleAndUtilization` to `profiles` to rebalance
    workers;
  - `AffinityAndTaints` already evicts pods that violate the infra taint, for
    example workloads left on the new infra nodes after they were tainted.
  Workloads tolerating the infra taint (routers, monitoring, registry) are not
  affected.
