# Operators (`baseline-operators`)

Day-2 platform operators installed on every cluster.

| Policy | Mode | What it does |
|---|---|---|
| `day2-gatekeeper-operator` | enforce | Gatekeeper Operator (`gatekeeper-operator-product`, `redhat-operators`) in `openshift-operators`, then a `Gatekeeper` instance: audit 1 replica, validating webhook on (3 replicas), **mutating webhook off**. |
| `day2-descheduler-operator` | enforce | Kube Descheduler Operator in `openshift-kube-descheduler-operator`, then `KubeDescheduler/cluster`: mode **`Predictive`** (evaluates and reports evictions, evicts nothing), profiles `AffinityAndTaints` and `TopologyAndDuplicates`, every 3600s. |

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

- **Gatekeeper:** the webhook runs **3 replicas**, one per node on the 3
  schedulable nodes. Some Gatekeeper webhooks use `failurePolicy: Fail`, so
  admission must survive the loss of a node. Its pods belong on worker nodes;
  on the current hub (no workers) they run on the infra nodes.
- **Descheduler:** `TopologyAndDuplicates` reports pods that break a hard
  topology spread constraint or share a node with another replica of the same
  owner. On the hub it flags the image registry, which a rollout left with 2
  of its 3 pods on one infra node. It still evicts nothing in `Predictive`
  mode:
  - after reviewing the Predictive results, set `mode: Automatic`;
  - `AffinityAndTaints` evicts pods that violate taints or node affinity, for
    example workloads left on infra nodes after they were tainted.
