# Compliance (`baseline-compliance`)

Installs the Compliance Operator and scans every cluster against **PCI-DSS
v4.0**, the profile set that fits the banking sector. Scan results are surfaced
in ACM.

| Policy | Mode | What it does |
|---|---|---|
| `day2-compliance-operator` | enforce | Namespace `openshift-compliance` (pod security `privileged`, the scanner pods need it); Compliance Operator from `redhat-operators`, channel `stable`, OperatorGroup scoped to its own namespace; ScanSettingBinding `pci-dss`. |
| `day2-compliance-pci-dss-results` | inform | Depends on the policy above. Compliant only when the `pci-dss` ComplianceSuite is `DONE` **and** has no `FAIL` check results. |

## The scan

| | |
|---|---|
| Profiles | `ocp4-pci-dss-4-0` (platform/API checks), `ocp4-pci-dss-node-4-0` (checks on every node) |
| ScanSetting | `default`: daily at 01:00, node roles `master` and `worker`, tolerates all taints, raw results kept on a PVC from the default StorageClass |
| Remediation | **Not automatic.** Many remediations are MachineConfigs that reboot nodes. Review them, then apply selectively. |

`day2-compliance-pci-dss-results` is **expected to stay NonCompliant** until the
failing checks are remediated or accepted. Useful commands:

```
oc get compliancesuite,compliancescan -n openshift-compliance
oc get compliancecheckresult -n openshift-compliance \
  -l compliance.openshift.io/suite=pci-dss,compliance.openshift.io/check-status=FAIL
oc get complianceremediation -n openshift-compliance
# apply one remediation:
oc patch complianceremediation <name> -n openshift-compliance --type merge -p '{"spec":{"apply":true}}'
```

Some checks are reported as `MANUAL`: they need an organizational or
procedural answer and are not counted as `FAIL`.

## Current setup (single node)

The operator is installed and the scan is scheduled, but on the hub the scan is
**stuck**: the single node has reached its limit of 250 pods per node, so the 8
scanner and aggregator pods stay `Pending` (`Too many pods`). Free pods or raise
`maxPods` with a KubeletConfig to let the scan run.

## Required changes for 3 infra nodes

- **The scan runs as is.** The default ScanSetting tolerates every taint, and
  the per-node pod limit is no longer a constraint. Infra nodes are scanned by
  the `worker` node scan as long as they still carry the
  `node-role.kubernetes.io/worker` label.
- **Recommended, and required if the worker label was removed from the infra
  nodes:** scan the infra nodes as their own role. Create a ScanSetting
  (for example `pci-dss`) with
  ```yaml
  roles: [master, worker, infra]
  ```
  and point `settingsRef` of the ScanSettingBinding in
  `day2-compliance-operator.yaml` at it. Node remediations then target the
  `infra` MachineConfigPool as well.
- When applying node remediations, nodes reboot one pool at a time. With 3
  infra nodes the routers and monitoring stay available while each node
  restarts.
