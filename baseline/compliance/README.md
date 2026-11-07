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
| ScanSetting | `pci-dss` (created by the policy, same values as `default`): daily at 01:00, tolerates all taints, raw results kept on a PVC from the default StorageClass. Node roles are read from each cluster: `master`, plus `worker` if there are worker nodes, plus `infra` if any infra node lacks the worker label |
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

## Current setup (3 control-plane + 3 infra nodes)

The hub has no worker nodes, and its infra nodes carry only the
`node-role.kubernetes.io/infra` label. With the `default` ScanSetting the
`worker` node scan was `NOT-APPLICABLE` and only the 3 masters were checked.
The `pci-dss` ScanSetting renders `roles: [master, infra]` there, so all 6
nodes are scanned and node remediations target the `infra` MachineConfigPool
as well.

When applying node remediations, nodes reboot one pool at a time. With 3 infra
nodes the routers and monitoring stay available while each node restarts.

```
oc get scansetting pci-dss -n openshift-compliance -o jsonpath='{.roles}'
oc get compliancescan -n openshift-compliance   # ...-node-4-0-master and ...-node-4-0-infra
```
