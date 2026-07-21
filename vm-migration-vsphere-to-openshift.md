# VM Migration Guide: VMware vSphere → Red Hat OpenShift

Full step-by-step guide for migrating VMs from VMware vSphere to Red Hat OpenShift Virtualization using the Migration Toolkit for Virtualization (MTV / Forklift operator).

## Migration Architecture

```
VMware vSphere (Source VMs)  --VDDK-->  Migration Toolkit for Virtualization (MTV)  --KubeVirt-->  Red Hat OpenShift (Target Cluster)
```

vSphere VMs are transferred via VDDK by MTV, then converted to KubeVirt VMs on OpenShift.

## Scope & Environment

| Item | Detail |
|---|---|
| Scope | 2–3 VMs manual migration, then Ansible automation for remaining VMs |
| Goal | Build proficiency for large-scale migration (up to 200 VMs in a single migration window) |
| Environment | UAT OpenShift Cluster + VMware vSphere |
| Tool | Migration Toolkit for Virtualization (MTV) — Forklift operator on OpenShift |
| Migration Type | Cold migration (VM powered off) for initial test — warm available after validation |

> Web UI steps are the primary method; CLI commands are provided as a supplement for verification and troubleshooting.

---

## Phase 1 — Verify vSphere Access

**1.1 Log into vCenter Web UI**

Confirm access to the vSphere environment and locate the VMs to migrate.

1. Open `https://<vcenter-ip-or-hostname>/ui`
2. Accept the self-signed certificate warning if prompted
3. Log in with vCenter admin credentials

```bash
# Confirm vCenter is reachable
curl -k https://<vcenter-ip>

# Confirm port 443 is open
nc -zv <vcenter-ip> 443
```

**1.2 Locate the Target VMs**

Navigate: **Menu → Hosts and Clusters → [Datacenter] → [Cluster] → VMs tab**

For each target VM, note down: VM name, current power state, vCPUs, RAM, disk sizes, and network/portgroup — these are needed later for the MTV migration plan and network/storage maps.

---

## Phase 2 — Verify OpenShift Cluster Access

**2.1 Log into the OpenShift Console**

Confirm the target OpenShift cluster is healthy before starting migration work — a degraded cluster causes migration failures.

Check: Control Plane is Healthy, all Cluster Operators are Available, all Nodes are Ready. If anything is degraded, escalate before proceeding.

```bash
oc login --token=sha256~<your-token> --server=https://<api-url>:6443
oc whoami

# Check nodes and operators
oc get nodes
oc get co | grep -v Available

# List available storage classes
oc get storageclass
```

**2.2 Create the Migration Namespace**

```bash
oc new-project uat-migration
oc get project uat-migration
```

**2.3 Grant MTV Service Account Permissions**

```bash
oc adm policy add-role-to-user \
  edit \
  system:serviceaccount:openshift-mtv:forklift-controller \
  -n uat-migration
```

> This allows the MTV `forklift-controller` to create and manage VM objects in the migration namespace.

---

## Phase 3 — Install / Verify MTV Operator

> ⚠️ On a cluster that's been idle for a while, assume the MTV operator may need reinstalling even if it shows a previous "Succeeded" status — verify all pods are actually running before trusting it.

**3.1 Check if MTV is already installed**

```bash
oc get ns | grep mtv
oc get csv -n openshift-mtv
oc get pods -n openshift-mtv
oc get pods -n openshift-mtv | grep -v Running
```

**3.2 Install MTV from OperatorHub (if not installed or needs reinstalling)**

1. OperatorHub → search "Migration Toolkit for Virtualization" → Install
2. Update channel: latest `release-v2.x`
3. Installation mode: a specific namespace → create `openshift-mtv`
4. Update approval: Automatic

```bash
# Verify all MTV pods are running after install
oc get pods -n openshift-mtv
# Expected: forklift-controller, forklift-must-gather-api, forklift-ui, forklift-validation, hook-runner — all Running

oc get csv -n openshift-mtv
# Expected: PHASE = Succeeded
```

**3.3 Access the MTV Console**

Application Launcher (grid icon) → Migration Toolkit for Virtualization → opens at `https://virt-<cluster-url>/mtv`

---

## Phase 4 — Build and Push the VDDK Image

VDDK (VMware Virtual Disk Development Kit) is VMware's proprietary library for reading VM disk data. MTV uses it to connect to vSphere and transfer disks. Red Hat cannot bundle VDDK directly — it must be downloaded from VMware and built into a container image, or migrations cannot start.

**4.1 Download VDDK from VMware**

Requires a VMware Customer Connect account. Download the version matching your vSphere version (7.x → VDDK 7.0.x, 8.x → VDDK 8.0.x) from `developer.vmware.com/web/sdk/8.0/vddk`.

```bash
ls -lh VMware-vix-disklib-*.tar.gz
tar tzf VMware-vix-disklib-*.tar.gz | head -20
```

**4.2 Create the VDDK Container Image**

```bash
mkdir vddk-build && cd vddk-build
tar xzf ../VMware-vix-disklib-*.tar.gz

cat > Dockerfile << 'EOF'
FROM registry.access.redhat.com/ubi8/ubi-minimal
COPY vmware-vix-disklib-distrib/ /vmware-vix-disklib-distrib/
RUN mkdir -p /opt
ENTRYPOINT ["cp", "-r", "/vmware-vix-disklib-distrib", "/opt"]
EOF

podman build -t vddk:latest .
podman images | grep vddk
```

**4.3 Push the VDDK Image to the Container Registry**

```bash
podman login <registry-url> -u <your-username> -p <your-password>
podman tag vddk:latest <registry-url>/openshift-mtv/vddk:latest
podman push <registry-url>/openshift-mtv/vddk:latest
```

**4.4 / 4.5 — Create Pull Secrets**

The pull secret must exist in **both** namespaces — `openshift-mtv` (for the MTV controller) and the migration target namespace (for the importer pod). Missing either causes the migration to fail when pulling the VDDK image.

```bash
oc create secret docker-registry vddk-registry-secret \
  --docker-server=<registry-url> \
  --docker-username=<registry-username> \
  --docker-password=<registry-password> \
  -n openshift-mtv

oc create secret docker-registry vddk-registry-secret \
  --docker-server=<registry-url> \
  --docker-username=<registry-username> \
  --docker-password=<registry-password> \
  -n uat-migration
```

---

## Phase 5 — Configure MTV Providers

A **Provider** defines a connection to either a source (vSphere) or target (OpenShift) environment. Two are needed — one per side.

**5.1 Verify the OpenShift Local Provider**

Usually auto-created on MTV install. MTV Console → Providers → confirm a provider named `host`/`local` shows **Connected**.

```bash
oc get providers -n openshift-mtv
oc describe provider -n openshift-mtv <provider-name>
```

**5.2 Create the vSphere Provider**

```bash
# Get the vCenter SSL fingerprint
openssl s_client -connect <vcenter-ip>:443 < /dev/null 2>/dev/null \
  | openssl x509 -fingerprint -noout -sha256
```

In MTV Console → Providers → Add Provider:
- Type: VMware
- vCenter host, username, password
- VDDK init image: the URL from Phase 4
- SHA-256 fingerprint: from the command above

```bash
# Confirm the provider connects
oc get providers -n openshift-mtv
# Expected: TYPE=vsphere, status Connected

oc describe provider vsphere-uat -n openshift-mtv | tail -30
```

---

## Phase 6 — Create Network and Storage Maps

**Network Map** translates a vSphere port group to an OpenShift network. **Storage Map** translates a vSphere datastore to an OpenShift storage class. At least one of each is required before creating a migration plan.

```bash
# Check what's available first
oc get storageclass
oc get network-attachment-definitions -A
```

**Network Map:** MTV Console → NetworkMaps → Create → map the vSphere port group (e.g. `VM-Network`) to the OpenShift target (e.g. Pod Networking, or a specific NAD if VLAN-tagged).

**Storage Map:** MTV Console → StorageMaps → Create → map the vSphere datastore to an OpenShift storage class (e.g. `ocs-storagecluster-ceph-rbd` for block storage).

```bash
oc get networkmaps -n openshift-mtv
oc get storagemaps -n openshift-mtv
# Both should show status Ready
```

---

## Phase 7 — Create and Run the Migration Plan

> Always migrate a single VM first as a proof of concept, validating VDDK connectivity, network/storage mapping, and the full conversion process before migrating the rest. Use Cold migration for the first test.

**7.1 Create the Migration Plan**

MTV Console → Migration Plans → Create Plan:
1. General settings — plan name, source provider, target provider, target namespace
2. Select VMs — start with just 1
3. Select the Network Map
4. Select the Storage Map
5. Migration type — Cold
6. Hooks — optional, skip for now
7. Review and Finish

**7.2 Start the Migration**

Click **Start** on the plan. It progresses through these stages:

1. PreHook (if configured)
2. **AllocateDisks** — creates PVCs in OpenShift for each disk
3. **CopyDisks** — transfers disk data via VDDK (longest step)
4. **Cutover** — powers off the source VM (cold) or performs final sync (warm)
5. **ConvertImage** — converts VMDK disks to KubeVirt-compatible format
6. **CreateVM** — creates the VirtualMachine object in OpenShift
7. PostHook (if configured)
8. Completed

```bash
# Watch the migration plan
oc get plans -n openshift-mtv -w
oc get migrations -n openshift-mtv

# Check PVCs being created (should show Bound during CopyDisks)
oc get pvc -n uat-migration

# Check importer pod logs
oc get pods -n uat-migration
oc logs -n uat-migration <importer-pod-name> --follow

# Check forklift controller for errors
oc logs -n openshift-mtv $(oc get pods -n openshift-mtv -l app=forklift-controller -o name) --follow

# If stuck, check namespace events
oc get events -n uat-migration --sort-by='.lastTimestamp' | tail -20
```

---

## Phase 8 — Post-Migration Validation

**8.1 Verify the VM Appears in OpenShift Virtualization**

Console → Virtualization → VirtualMachines → select the migration namespace → confirm the VM appears (initially **Stopped** after a cold migration).

**8.2 Review Configuration Before Starting**

Compare CPU, memory, disks, and network interfaces against the original vSphere VM noted in Phase 1. If anything looks wrong, investigate the migration logs before powering on.

**8.3 Power On and Access the VM**

Actions → Start → wait for **Running** → open the Console tab → log in and confirm the OS boots cleanly.

**8.4 Validate Network and Application**

```bash
# From inside the VM
ip addr show
ip route
ping -c 4 <gateway-ip>
systemctl list-units --state=failed
df -h

# From OpenShift CLI
oc get vms -n uat-migration
oc get vmi -n uat-migration
oc describe vm <vm-name> -n uat-migration
oc get events -n uat-migration | grep <vm-name>
oc get pvc -n uat-migration
```

**8.5 Validation Checklist**

- VM appears in OpenShift Virtualization, in the correct namespace
- CPU and memory match the source
- All disks present with correct sizes
- Network interface present and connected to the mapped network
- VM powers on without errors, OS boots cleanly
- IP address assigned, gateway ping successful, SSH access works
- No failed services, all expected disk mounts present
- Applications/APIs responding as expected

Only after every item is confirmed should the next VM be migrated — repeat Phases 7–8 for each remaining VM.

---

## Phase 9 — Troubleshooting Common Issues

**Provider stuck at "Not Ready"**

```bash
oc describe provider vsphere-uat -n openshift-mtv | tail -40
```

Common causes: wrong VDDK image URL, wrong vCenter credentials, SSL fingerprint mismatch, or network connectivity blocked between the OpenShift worker nodes and vCenter.

```bash
# Test connectivity directly from a worker node
oc debug node/<worker-node-name>
curl -k https://<vcenter-ip>
```

**Migration stuck at CopyDisks**

```bash
oc get pods -n uat-migration | grep importer
oc logs -n uat-migration <importer-pod-name> --follow
```

Common causes: VDDK image pull failure, insufficient storage in the target namespace, or a vSphere connectivity timeout.

**VM fails to boot after migration**

- **Windows VMs:** missing VirtIO drivers — boot from the VirtIO ISO and install drivers
- **Network interface renamed** (e.g. `eth0` → `ens3`): update the network config from the console
- **Disk UUID changed, breaking fstab:** update `/etc/fstab` to use device names instead of UUIDs

```bash
# Access the VM console from CLI if the UI console isn't working
virtctl console <vm-name> -n uat-migration
```

**Useful log locations**

| Component | Command |
|---|---|
| MTV Forklift Controller | `oc logs -n openshift-mtv $(oc get pods -n openshift-mtv -l app=forklift-controller -o name)` |
| MTV Validation Service | `oc logs -n openshift-mtv $(oc get pods -n openshift-mtv -l app=forklift-validation -o name)` |
| Disk Importer Pod | `oc logs -n uat-migration <importer-pod-name>` |
| VM Events | `oc get events -n uat-migration --sort-by=.lastTimestamp` |
| Provider Events | `oc describe provider vsphere-uat -n openshift-mtv` |

---

## Lessons Learned

- Always validate with a single VM cold migration before scaling to a full batch — it surfaces VDDK, network map, and storage map issues early, when they're cheap to fix.
- The VDDK image and its pull secrets are the most common early blocker — the secret must exist in **both** the MTV namespace and the target migration namespace.
- Windows VM boot failures after migration almost always trace back to missing VirtIO drivers, not a bad migration — this is a source-side fix, not a target-side one.
- Network interface renaming and fstab UUID mismatches are common but recoverable directly from the VM console — no need to re-migrate.

---
*Documented from hands-on VM migration work using the Migration Toolkit for Virtualization (MTV) in an OpenShift Virtualization environment.*
