# OpenShift Virtualization Expansion — Deployment Runbook

## Environment Summary

| Component | Count | Role |
|---|---|---|
| Control plane nodes | 3 | Cluster brain |
| Infra nodes | 3 | ODF storage |
| Existing worker nodes | 6 | Container workloads |
| New worker nodes | 7 | VM workloads only |

> All `oc` commands run from the bastion host unless stated otherwise.

---

## Chapter 1 — Check Current Cluster Health Before Adding Nodes

**Run from:** Bastion host

**What this does:** Before adding any new nodes, confirm the existing cluster is in a healthy state. You are checking that all nodes are up, all cluster services are running fine, the cluster is not mid-upgrade, and etcd (the brain of the cluster) is healthy. If anything is broken here, fix it before proceeding.

```bash
oc get nodes
# Confirms all existing nodes are in Ready state.

oc get co
# Checks all cluster operators are Available and not Degraded or Progressing.

oc get clusterversion
# Confirms the cluster is not mid-upgrade and is on a stable version.

oc get etcd -o=jsonpath='{range .items[0].status.conditions[*]}{.type}{"\t"}{.status}{"\n"}{end}'
# Checks etcd health. etcd stores all cluster state — it must be healthy before adding nodes.
```

**Expected output:**
- All nodes showing Ready
- All cluster operators showing AVAILABLE=True, PROGRESSING=False, DEGRADED=False
- ClusterVersion showing AVAILABLE=True, PROGRESSING=False
- etcd conditions all returning True

---

## Chapter 2 — Generate Ignition Config for New Worker Nodes (Assisted Installer Method)

**Run from:** Browser (console.redhat.com)

**What this does:** The ignition config tells each new node everything it needs to join the cluster. When using the Assisted Installer, this file is automatically embedded inside the discovery ISO you download. By downloading the ISO, you already have the ignition config — nothing to generate manually.

1. Go to `https://console.redhat.com/openshift`
2. Find the cluster → click **Add hosts**
3. Select architecture: `x86_64`
4. Configure static network settings for the machine network bond if prompted
5. Click **Generate Discovery ISO** → **Download ISO**

**Expected output:**
- ISO file downloaded successfully
- The `worker.ign` file is already embedded inside the ISO — no separate action needed

Keep the ISO on your bastion host or somewhere reachable over the network. You will use it in the next chapter.

---

## Chapter 3 — Boot the 7 Nodes with RHCOS ISO and Join the Cluster

**Run from:** iDRAC Virtual Media (booting) + Bastion host (CSR approval)

**What this does:** You remotely mount the ISO on each of the 7 servers using iDRAC Virtual Media and boot them from it. Once each server boots, it reads the embedded ignition config, installs RHCOS, and registers itself with the cluster and on console.redhat.com. The nodes then send certificate requests (CSRs) asking to join the cluster. You approve the first round — called bootstrapper CSRs — and watch the nodes appear.

**Boot each server via iDRAC Virtual Media — repeat for all 7:**

1. Open `https://<IDRAC_IP>` in your browser
2. Log in with iDRAC credentials
3. Go to Configuration → Virtual Media
4. Click Connect Virtual Media
5. Select the RHCOS ISO file from the jump server
6. Reboot the server and set boot device to Virtual CD/DVD

Once nodes start booting, switch to the bastion and check if nodes are appearing:

```bash
oc get nodes
# You will start seeing the new nodes listed — they will show as NotReady at this stage. That is normal.
```

Check for incoming CSRs:

```bash
oc get csr
# You will see pending CSRs. The first round are called bootstrapper CSRs.

oc get csr | grep node-bootstrapper
# Filter to see only the bootstrapper CSRs.
```

Approve each bootstrapper CSR one by one — replace `csr-####` with the actual name from the list:

```bash
oc adm certificate approve csr-####
```

Run `oc get csr` again after approving to confirm each one moves from Pending to Approved.

```bash
oc get nodes -w
# Watch the nodes appear in the cluster.
```

**Expected output:**
- All 7 nodes appear in the cluster with status NotReady
- All bootstrapper CSRs move from Pending to Approved
- Nodes are visible on console.redhat.com under the cluster

> The nodes will stay NotReady until you approve the second round of CSRs in Chapter 5. This is expected — do not worry.

---

## Chapter 4 — Taint the 7 New Nodes

**Run from:** Bastion host

**What this does:** You label and taint the 7 new nodes so only VM workloads run on them. The label identifies them as virtualization nodes and the taint blocks any regular container workloads from scheduling on them. Be very careful to only apply this to the 7 new nodes.

Before you start — confirm the 7 new nodes are showing in the cluster. They will be in NotReady state at this point. That is fine and expected:

```bash
oc get nodes
# You should see your 7 new nodes listed alongside the existing 12 nodes. The new ones will show NotReady.
```

Label all 7 nodes — replace with each actual node name:

```bash
oc label node <node-name> node-role.kubernetes.io/virt-worker="" workload-type=virtualization
# Repeat for each of the 7 nodes one by one.
```

Taint all 7 nodes — replace with each actual node name:

```bash
oc adm taint node <node-name> workload=virtualization:NoSchedule
# Repeat for each of the 7 nodes one by one.
```

Verify taints are applied:

```bash
oc describe nodes -l node-role.kubernetes.io/virt-worker | grep -A 3 Taints
```

**Expected output:**
- All 7 nodes show label `node-role.kubernetes.io/virt-worker`
- All 7 nodes show taint `workload=virtualization:NoSchedule`
- No existing nodes affected

---

## Chapter 5 — Approve Second Round of CSRs and Confirm Nodes are Ready

**Run from:** Bastion host

**What this does:** After approving the bootstrapper CSRs in Chapter 3, the nodes send a second round of CSRs to complete their registration into the cluster. You approve these, watch the nodes go from NotReady to Ready, then confirm no regular container workloads have landed on the virt nodes.

```bash
oc get csr | grep Pending
# You will see new CSRs in Pending state — these are the second round. If nothing shows yet, wait a minute and run it again.

oc adm certificate approve csr-####
# Replace csr-#### with the actual name. Run oc get csr again to confirm each moves from Pending to Approved.

oc get nodes -w
# Watch nodes transition to Ready. Expected: All 7 nodes transition from NotReady to Ready.

oc get pods -A -o wide | grep virt-node
# Confirm no non-system pods scheduled on virt nodes. Expected: No regular container workloads on any of the 7 virt nodes.
```

---

## Chapter 6 — Create a New MachineConfigPool for the Virt Nodes

**Run from:** Bastion host

**What this does:** A MachineConfigPool groups nodes together so configuration changes only apply to them. By creating a dedicated pool called `virt-workers`, any changes you make later only affect the 7 new virt nodes and never touch the existing worker nodes.

```yaml
cat << EOF | oc apply -f -
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfigPool
metadata:
  name: virt-workers
  labels:
    machineconfiguration.openshift.io/role: virt-workers
spec:
  machineConfigSelector:
    matchExpressions:
      - key: machineconfiguration.openshift.io/role
        operator: In
        values:
          - worker
          - virt-workers
  nodeSelector:
    matchLabels:
      node-role.kubernetes.io/virt-worker: ""
  paused: false
EOF
```

```bash
oc get mcp
oc describe mcp virt-workers | grep "Machine Count"
```

**Expected output:**
- `virt-workers` pool appears in the list
- Machine Count shows 7
- UPDATED=True, DEGRADED=False

---

## Chapter 7 — Create Custom MachineConfig for Virt Nodes

**Run from:** Bastion host

**What this does:** Applies custom virtualization tuning to the 7 virt nodes. Sets kernel arguments and system settings required for running VMs efficiently. Each node will reboot automatically to apply the new configuration.

Kernel arguments being applied:
- `intel_iommu=on` — Required for PCI passthrough to VMs
- `iommu=pt` — Passthrough mode for better performance
- `kvm-intel.nested=1` — Enables nested virtualisation

```yaml
cat << EOF | oc apply -f -
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  name: 50-virt-workers-config
  labels:
    machineconfiguration.openshift.io/role: virt-workers
spec:
  kernelArguments:
    - intel_iommu=on
    - iommu=pt
    - kvm-intel.nested=1
  config:
    ignition:
      version: 3.2.0
    storage:
      files:
        - path: /etc/sysctl.d/99-virt-tuning.conf
          mode: 0644
          contents:
            source: data:,vm.swappiness%3D10%0Akernel.numa_balancing%3D1
    systemd:
      units:
        - name: kubevirt-tuning.service
          enabled: true
          contents: |
            [Unit]
            Description=KubeVirt performance tuning
            After=network.target
            [Service]
            Type=oneshot
            ExecStart=/bin/bash -c 'echo never > /sys/kernel/mm/transparent_hugepage/enabled'
            RemainAfterExit=yes
            [Install]
            WantedBy=multi-user.target
EOF
```

Watch the rollout — each node will reboot:

```bash
watch oc get mcp virt-workers
oc get nodes -l node-role.kubernetes.io/virt-worker -w
```

**Expected output:**
- MCP shows UPDATING=True while nodes reboot
- Once done: UPDATED=True, DEGRADED=False
- All 7 nodes back to Ready

---

## Chapter 8 — Verify ODF Storage is Healthy

**Run from:** Bastion host

**What this does:** ODF is the storage backend for all VM disks. Confirm it is fully healthy before installing OpenShift Virtualization. If ODF has any warnings or errors, fix them first — do not proceed with a degraded storage cluster.

```bash
oc get nodes -l cluster.ocs.openshift.io/openshift-storage="" -o wide
oc get storagecluster -n openshift-storage
oc get storagecluster ocs-storagecluster -n openshift-storage -o jsonpath='{.status.phase}'
oc get storageclass | grep openshift-storage
oc get pods -n openshift-storage | grep -v Running | grep -v Completed
oc rsh -n openshift-storage $(oc get pods -n openshift-storage -l app=rook-ceph-tools -o jsonpath='{.items[0].metadata.name}') ceph status
oc rsh -n openshift-storage $(oc get pods -n openshift-storage -l app=rook-ceph-tools -o jsonpath='{.items[0].metadata.name}') ceph health detail
```

**Expected output:**
- StorageCluster phase returns Ready
- No pods in error or pending state
- Ceph status shows HEALTH_OK

> ⚠️ Do not proceed if Ceph shows HEALTH_WARN or HEALTH_ERR. Fix ODF first.

---

## Chapter 9 — Connect New Virt Nodes to ODF

**Run from:** Bastion host

**What this does:** The 7 virt nodes need to reach the ODF storage cluster on the infra nodes. You test that virt nodes can communicate with Ceph over the network and verify the CSI storage driver pods are running on all 7 virt nodes. The CSI driver is what allows VMs to claim and use storage from ODF.

```bash
oc get storageclass | grep ocs
oc get endpoints -n openshift-storage | grep mon
```

Test Ceph MON reachability from a virt node:

```bash
oc debug node/virt-node-1 -- \
  chroot /host bash -c \
  "curl -k https://rook-ceph-mon-a.openshift-storage.svc:3300"
# Repeat for each virt node.
```

```bash
oc get pods -n openshift-storage -l app=csi-rbdplugin -o wide | grep virt-node
oc get pods -n openshift-storage -l app=csi-cephfsplugin -o wide | grep virt-node
```

**Expected output:**
- OCS storage classes visible
- Ceph MON reachable from all virt nodes
- CSI RBD and CephFS pods running on all 7 virt nodes

> ⚠️ If CSI pods are not running on virt nodes, the CSI daemonsets need a toleration for `workload=virtualization:NoSchedule`. Patch them before proceeding.

---

## Chapter 10 — Install OpenShift Virtualization and Configure HyperConverged

**Run from:** Bastion host + OCP Web Console (for operator install)

**What this does:** Verify whether OpenShift Virtualization is already installed. If not, install it from OperatorHub. Then configure the HyperConverged object to run exclusively on the 7 virt nodes.

Check if already installed:

```bash
oc get csv -n openshift-cnv | grep kubevirt
# Expected: Returns Succeeded — if nothing returns, install it.
```

If not installed — install via OCP Web Console:

1. Log into the OCP web console
2. Go to Operators → OperatorHub
3. Search for **OpenShift Virtualization**
4. Click Install → select namespace `openshift-cnv`
5. Wait for CSV to show Succeeded

Create the HyperConverged object — pins all virtualization components to the 7 virt nodes only:

```yaml
cat << EOF | oc apply -f -
apiVersion: hco.kubevirt.io/v1beta1
kind: HyperConverged
metadata:
  name: kubevirt-hyperconverged
  namespace: openshift-cnv
spec:
  infra:
    nodePlacement:
      nodeSelector:
        node-role.kubernetes.io/virt-worker: ""
      tolerations:
        - key: workload
          operator: Equal
          value: virtualization
          effect: NoSchedule
  workloads:
    nodePlacement:
      nodeSelector:
        node-role.kubernetes.io/virt-worker: ""
      tolerations:
        - key: workload
          operator: Equal
          value: virtualization
          effect: NoSchedule
EOF
```

```bash
oc get hco -n openshift-cnv
oc get pods -n openshift-cnv -o wide
```

**Expected:** HyperConverged shows Available. All CNV pods on virt nodes only.

> ⚠️ When creating any VM, always include this nodeSelector and toleration so it schedules on virt nodes only:
> ```yaml
> spec:
>   template:
>     spec:
>       nodeSelector:
>         node-role.kubernetes.io/virt-worker: ""
>       tolerations:
>         - key: workload
>           operator: Equal
>           value: virtualization
>           effect: NoSchedule
> ```

---

## Chapter 11 — Configure OpenShift Virtualization to Use ODF Storage

**Run from:** Bastion host

**What this does:** Sets ODF RBD as the default storage class for VMs so any VM created automatically uses ODF storage without needing to specify it each time.

```bash
oc patch storageclass ocs-storagecluster-ceph-rbd \
  --type=merge \
  -p '{"metadata": {"annotations": {
    "storageclass.kubernetes.io/is-default-class": "true",
    "storageclass.kubevirt.io/is-default-virt-class": "true"
  }}}'

oc get storageclass
```

**Expected:** `ocs-storagecluster-ceph-rbd` shows (default).

---

## Chapter 12 — Configure HyperConverged to Use ODF Storage

**Run from:** Bastion host

**What this does:** Configures HyperConverged to use ODF as the backend for VM image imports and sets up an automated job that pulls the RHEL9 base image from Red Hat into ODF so it is ready when creating VMs.

```bash
oc edit hyperconverged kubevirt-hyperconverged -n openshift-cnv
```

Add the following under `spec`:

```yaml
spec:
  storageImport:
    insecureRegistries: []
  dataImportCronTemplates:
    - metadata:
        name: rhel9-image-cron
      spec:
        template:
          spec:
            storage:
              storageClassName: ocs-storagecluster-ceph-rbd
              accessModes:
                - ReadWriteMany
              resources:
                requests:
                  storage: 30Gi
```

```bash
oc get hyperconverged kubevirt-hyperconverged -n openshift-cnv -o yaml | grep -A 10 dataImportCronTemplates
```

**Expected:** `rhel9-image-cron` template is visible and configured correctly.

---

## Chapter 13 — Configure Virtualization Network (NMState + NAD)

**Run from:** Bastion host

**What this does:** Configures the physical network on the 7 virt nodes by creating an LACP bond from the two virtualization NICs and then creating a Linux bridge on top of that bond. The bridge is what VMs will connect to. A NetworkAttachmentDefinition (NAD) is then created to expose the bridge to VMs so they can attach to it and communicate on the network.

### Step 1 — Test bond on a single node first

Always test on one node before rolling out to all 7.

```yaml
cat << EOF | oc apply -f -
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: vm-test-bond-worker11
spec:
  nodeSelector:
    kubernetes.io/hostname: <worker-node-hostname>
  desiredState:
    interfaces:
      - name: bond-vm
        type: bond
        state: up
        link-aggregation:
          mode: 802.3ad
          port:
            - eno12409np1
            - ens5f0np0
EOF
```

Verify bond is working on that node:

```bash
oc get nnce | grep vm-test-bond-worker11
oc debug node/<worker-node-hostname>
cat /proc/net/bonding/bond-vm
ip link show bond-vm
```

**Expected:** Bond shows both ports active and LACP mode confirmed.

### Step 2 — Test bond + bridge on the same single node

```yaml
cat << EOF | oc apply -f -
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: vm-test-bridge-worker11
spec:
  nodeSelector:
    kubernetes.io/hostname: <worker-node-hostname>
  desiredState:
    interfaces:
      - name: bond-vm
        type: bond
        state: up
        link-aggregation:
          mode: 802.3ad
          port:
            - eno12409np1
            - ens5f0np0
      - name: br-vm
        type: linux-bridge
        state: up
        bridge:
          options:
            stp:
              enabled: false
          port:
            - name: bond-vm
EOF
```

Verify bridge is on the node:

```bash
oc get nns <worker-node-hostname> -o yaml | grep br-vm
ip link show br-vm
```

**Expected:** Shows `name: br-vm` and `controller: br-vm`.

### Step 3 — Roll out bond + bridge to all 7 virt nodes

```yaml
cat << EOF | oc apply -f -
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: vm-network
spec:
  nodeSelector:
    node-role.kubernetes.io/virt-worker: ""
  desiredState:
    interfaces:
      - name: bond-vm
        type: bond
        state: up
        link-aggregation:
          mode: 802.3ad
          port:
            - eno12409np1
            - ens5f0np0
      - name: br-vm
        type: linux-bridge
        state: up
        bridge:
          options:
            stp:
              enabled: false
          port:
            - name: bond-vm
EOF
```

Watch the rollout:

```bash
oc get nncp
oc get nnce
```

**Expected:** All 7 virt nodes show Available=True.

Verify `br-vm` exists on all 7 nodes:

```bash
for n in $(oc get nodes -l node-role.kubernetes.io/virt-worker= -o name | cut -d/ -f2); do
  echo "===== $n ====="
  oc get nns $n -o yaml | grep "name: br-vm"
done
```

**Expected:** Every node returns `name: br-vm`.

### Step 4 — Create NetworkAttachmentDefinition (NAD)

**What this does:** Exposes the `br-vm` bridge to VMs so they can attach to it. Two NADs are created — one without a VLAN for general use and one with VLAN 216 for tagged traffic.

NAD without VLAN:

```yaml
cat << EOF | oc apply -f -
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: vm-network
  namespace: openshift-cnv
spec:
  config: '{
    "cniVersion": "0.3.1",
    "name": "vm-network",
    "type": "cnv-bridge",
    "bridge": "br-vm",
    "spoofChk": "off",
    "macspoofchk": false
  }'
EOF
```

NAD with VLAN 216:

```yaml
cat << EOF | oc apply -f -
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: vm-network-vlan216
  namespace: openshift-cnv
spec:
  config: '{
    "cniVersion": "0.3.1",
    "name": "vm-network-vlan216",
    "type": "cnv-bridge",
    "bridge": "br-vm",
    "vlan": 216
  }'
EOF
```

Confirm both NADs are created:

```bash
oc get network-attachment-definitions -n openshift-cnv
```

**Expected:** Both `vm-network` and `vm-network-vlan216` appear in the list.

### Step 5 — Test end to end connectivity

Attach the NAD to a VM and assign a static IP inside the VM:

```bash
# Inside the VM console
nmcli connection modify "Wired connection 1" ipv4.addresses <VM-IP>/<PREFIX> ipv4.gateway <GATEWAY-IP> ipv4.method manual
nmcli connection up "Wired connection 1"
```

Confirm IP is assigned:

```bash
ip addr show eth1
```

Ping the gateway to confirm connectivity:

```bash
ping -c 4 <GATEWAY-IP>
```

**Expected:** Ping responds successfully — VM is connected to the network end to end.

---

## Chapter 14 — Test a New VM Using ODF Storage

**Run from:** Bastion host

**What this does:** Deploy a test VM using an ODF-backed disk to confirm the full stack works end to end. The VM must schedule on a virt node and claim storage from ODF successfully. Clean up after verification.

```yaml
cat << EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-vm-odf-pvc
  namespace: default
spec:
  storageClassName: ocs-storagecluster-ceph-rbd
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  volumeMode: Block
---
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: test-odf-vm
  namespace: default
spec:
  running: true
  template:
    spec:
      nodeSelector:
        node-role.kubernetes.io/virt-worker: ""
      tolerations:
        - key: workload
          operator: Equal
          value: virtualization
          effect: NoSchedule
      domain:
        devices:
          disks:
            - name: testdisk
              disk:
                bus: virtio
        resources:
          requests:
            memory: 1Gi
      volumes:
        - name: testdisk
          persistentVolumeClaim:
            claimName: test-vm-odf-pvc
EOF
```

Watch the VM start:

```bash
oc get vmi test-odf-vm -w
```

Verify it landed on a virt node:

```bash
oc get vmi test-odf-vm -o jsonpath='{.status.nodeName}'
```

**Expected:** Returns one of your 7 virt node names.

Clean up:

```bash
oc delete vm test-odf-vm
oc delete pvc test-vm-odf-pvc
```

---

## Chapter 15 — Confirm All Remaining Operators Are Installed

**Run from:** Bastion host + OCP Web Console

**What this does:** Confirms all remaining required operators are installed and running.

```bash
oc get csv -A | grep -E "nmstate|metallb|node-maintenance|descheduler"
```

**Expected:** All return Succeeded. If any are missing, install via Operators → OperatorHub in the OCP web console.

| Operator | Namespace | Purpose |
|---|---|---|
| NMState Operator | openshift-nmstate | Declarative network config for nodes |
| MetalLB Operator | metallb-system | Load balancer IPs for VM services |
| Node Maintenance Operator | openshift-operators | Safely drain a node before maintenance |
| KubeDescheduler Operator | openshift-kube-descheduler-operator | Rebalance VMs across virt nodes |
| Machine Config Operator | Built into OCP — no install needed | Manages node config via MachineConfig |

After installing MetalLB, configure the IP pool and L2Advertisement:

```yaml
cat << EOF | oc apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: virt-ip-pool
  namespace: metallb-system
spec:
  addresses:
    - <METALLB_IP_RANGE_START>-<METALLB_IP_RANGE_END>
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: virt-l2-advert
  namespace: metallb-system
spec:
  ipAddressPools:
    - virt-ip-pool
EOF
```

> Replace `<METALLB_IP_RANGE_START>` and `<METALLB_IP_RANGE_END>` with the IP range from the network team.

```bash
oc get ipaddresspool -n metallb-system
oc get l2advertisement -n metallb-system
```

Final check — all operators succeeded:

```bash
oc get csv -A | grep -v Succeeded
```

**Expected:** No output.

---

## Chapter 16 — Compliance Operator

**Run from:** Bastion host + OCP Web Console

**What this does:** The Compliance Operator runs security checks against the cluster and VM workloads to ensure they meet security policies and standards. It continuously scans the environment and reports any violations.

Install via OCP Web Console:

1. Log into OCP web console
2. Go to Operators → OperatorHub
3. Search for **Compliance Operator**
4. Click Install → namespace `openshift-compliance`
5. Wait for CSV to show Succeeded

Confirm it is installed:

```bash
oc get csv -n openshift-compliance | grep compliance
```

Create a basic scan to confirm it is working:

```yaml
cat << EOF | oc apply -f -
apiVersion: compliance.openshift.io/v1alpha1
kind: ScanSettingBinding
metadata:
  name: cis-compliance
  namespace: openshift-compliance
spec:
  profiles:
    - name: ocp4-cis
      kind: Profile
      apiGroup: compliance.openshift.io/v1alpha1
  settingsRef:
    name: default
    kind: ScanSetting
    apiGroup: compliance.openshift.io/v1alpha1
EOF
```

Check scan results:

```bash
oc get compliancesuite -n openshift-compliance
oc get compliancecheckresult -n openshift-compliance | grep FAIL
```

**Expected output:**
- CSV shows Succeeded
- ComplianceSuite shows DONE
- Review any FAIL results and address them accordingly

---

## Chapter 17 — Monitoring Configuration

**Run from:** Bastion host

**What this does:** OpenShift Virtualization automatically installs its own monitoring rules and dashboards when the CNV operator is installed. You are just confirming they are active and working on the existing monitoring stack.

```bash
oc get servicemonitor -n openshift-cnv
oc get prometheusrule -n openshift-cnv
oc get pods -n openshift-monitoring | grep prometheus
```

Check Grafana dashboards via OCP web console:

1. Log into OCP web console
2. Go to Observe → Dashboards
3. Look for KubeVirt dashboards:
   - KubeVirt / Infrastructure Resources / Top Consumers
   - KubeVirt / Infrastructure Resources / VMs

**Expected output:**
- ServiceMonitors and PrometheusRules exist in `openshift-cnv`
- Prometheus pods are running
- KubeVirt dashboards visible in the console

---

## Chapter 18 — Logging Configuration

**Run from:** Bastion host

**What this does:** Confirms the existing cluster logging stack is picking up logs from the CNV components and VM workloads. Since this is an existing cluster, logging is already running — you just need to confirm CNV logs are being collected.

```bash
oc get clusterlogging -n openshift-logging
oc get pods -n openshift-logging
oc logs -n openshift-cnv $(oc get pods -n openshift-cnv -l app=virt-operator -o jsonpath='{.items[0].metadata.name}') | tail -20
```

**Expected output:**
- Cluster logging pods are running
- CNV logs are visible and being collected
- If external SIEM log forwarding is required, raise this as a separate task with the logging team

---

## Chapter 19 — Post Installation Validation

**Run from:** Bastion host

**What this does:** Final check to confirm everything installed and configured during this expansion is working correctly before sign off.

```bash
oc get csv -A | grep -v Succeeded
# Expected: No output — all operators in Succeeded state.

oc get nodes -l node-role.kubernetes.io/virt-worker
# Expected: All 7 nodes showing Ready.

oc get mcp virt-workers
# Expected: MACHINECOUNT=7, READYMACHINECOUNT=7, DEGRADED=False.

oc get pods -n openshift-cnv -o wide | grep -v virt-worker
# Expected: No output — CNV pods on virt nodes only.

oc get nncp
oc get nnce
# Expected: All nodes show Available=True.

oc get network-attachment-definitions -n openshift-cnv
# Expected: vm-network and vm-network-vlan216 both present.

oc get storagecluster -n openshift-storage
oc get cephcluster -n openshift-storage
# Expected: StorageCluster Ready, CephCluster HEALTH_OK.

oc get pods -n metallb-system
oc get ipaddresspool -n metallb-system
# Expected: All pods running, IP pool visible.
```

---

## Chapter 20 — Document Environment

**Run from:** Bastion host

**What this does:** Captures the final state of the expanded cluster as evidence that the work was completed successfully.

```bash
oc get nodes -o wide > /tmp/final-nodes.txt
oc get csv -A > /tmp/final-operators.txt
oc get mcp > /tmp/final-mcp.txt
oc get pods -n openshift-cnv -o wide > /tmp/final-cnv-pods.txt
oc get storageclass > /tmp/final-storageclass.txt
oc get network-attachment-definitions -n openshift-cnv > /tmp/final-nads.txt
oc get nncp > /tmp/final-nncp.txt
oc get ipaddresspool -n metallb-system > /tmp/final-metallb.txt
```

The final sign-off documentation should include:

1. Summary of work completed
2. List of all 7 new virt nodes with their hostnames and IPs
3. List of all operators installed and their versions
4. MachineConfigPool confirmation
5. ODF storage health status
6. Network configuration — bonds, bridges and NADs created
7. MetalLB IP pool configured
8. Screenshots of test VM running on virt node
9. Before and after node count
10. Any open items or known issues

Collect operator versions for the report:

```bash
oc get csv -n openshift-cnv | grep kubevirt
oc get csv -n openshift-nmstate | grep nmstate
oc get csv -n metallb-system | grep metallb
oc get csv -n openshift-operators | grep node-maintenance
```

**Expected output:**
- All output files saved to `/tmp/`
- Sign-off documentation updated with all the above information
- Customer signs off on the completed work

---

## Placeholder Reference

| Placeholder | Source |
|---|---|
| `<CLUSTER_NAME>` | `oc get infrastructure cluster` |
| `<BASE_DOMAIN>` | `oc get dns cluster` |
| `<METALLB_IP_RANGE_START/END>` | Network team |

---
*Documented from hands-on OpenShift Virtualization cluster expansion work as a support engineer on a 2-person delivery team.*
