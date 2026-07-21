# Red Hat Infrastructure Notes

Documentation from hands-on work across enterprise Red Hat environments — Linux administration, automation, and platform engineering.

Written from real troubleshooting and deployment work, not general tutorials — each guide includes the actual issues encountered and how they were resolved.

## Contents

- **[CentOS to RHEL Conversion Guide](./centos-to-rhel-conversion-guide.md)** — Step-by-step conversion process using `convert2rhel`, including Satellite registration setup and the SSL/CA certificate issue that blocks conversion in Satellite-managed environments.

- **[OpenShift Virtualization Expansion — Deployment Runbook](./openshift-virtualization-expansion.md)** — Full runbook for expanding an OpenShift cluster with dedicated virtualization worker nodes using the Assisted Installer method: iDRAC provisioning, CSR approval, MachineConfigPool setup, ODF storage integration, NMState networking (bonds/bridges/NADs), and HyperConverged configuration.

- **[VM Migration Guide: VMware vSphere → OpenShift](./vm-migration-vsphere-to-openshift.md)** — Step-by-step guide for migrating VMs using the Migration Toolkit for Virtualization (MTV), covering VDDK image builds, provider/network/storage map configuration, running a migration plan, post-migration validation, and troubleshooting common failures.

---

### About me

Red Hat Certified Linux & Automation Engineer (RHCE, RHCSA) working with RHEL, Ansible Automation Platform, and Red Hat Satellite — currently expanding into Openshift/kubernetes, OpenShift Virtualization and platform engineering.

[LinkedIn](https://www.linkedin.com/in/adedapoap) · [Profile README](https://github.com/AdedapoAduroja)
