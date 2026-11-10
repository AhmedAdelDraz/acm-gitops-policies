# Hub infrastructure (`hub-infra`)

Runs the hub's own infrastructure services on the infra nodes. Placed only on
the self-managed hub (`local-cluster=true`), the one cluster where ACM and the
hub's OpenShift GitOps run.

| Policy | Mode | What it does |
|---|---|---|
| `day2-hub-services-infra` | enforce | With infra nodes: `nodeSelector` and tolerations on the `MultiClusterHub` (the MCH operator passes them on to the `MultiClusterEngine`), `KlusterletConfig/global` (klusterlet agents of every managed cluster), `GitopsService/cluster` (`runOnInfra: true`, the default Argo CD instance), and the OLM Subscription `config` of the ACM, MCE and GitOps operators. Without infra nodes it only checks that the MultiClusterHub exists. |

**Tolerations:** every `NoSchedule` and `PreferNoSchedule` taint, plus any
effect of `node-role.kubernetes.io/infra`. Other `NoExecute` taints are not
tolerated, so pods are still rescheduled from a failed node.

**Not covered:**

- ACM add-on agents on `local-cluster` (`open-cluster-management-agent-addon`)
  are placed through AddOnDeploymentConfigs referenced by the
  ClusterManagementAddOns, which the MultiClusterHub operator owns.
- The MultiClusterEngine is not set directly: the MultiClusterHub operator owns
  its `nodeSelector` and `tolerations` and would overwrite them.

**Verify:**

```
oc get pods -n open-cluster-management -o wide
oc get pods -n multicluster-engine -o wide
oc get pods -n openshift-gitops -o wide
oc get multiclusterengine multiclusterengine -o jsonpath='{.spec.nodeSelector}{"\n"}{.spec.tolerations}'
```
