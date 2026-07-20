# CentOS → RHEL Conversion Guide

A step-by-step reference for converting CentOS Linux systems to Red Hat Enterprise Linux (RHEL) using the `convert2rhel` utility — documented from hands-on conversion work in an enterprise Red Hat environment.

> Run all commands as root or with sudo. Follow phases in order — each phase builds on the last.

---

## Phase 1 — Prerequisites & Preparation

**Verify system readiness.** CentOS Stream is NOT supported — only CentOS Linux.

```bash
# Check for CentOS Stream packages (CentOS 8 only)
rpm -qa | grep -i stream
yum remove <package-name>
```

**Stop services that could interfere or cause data corruption during conversion:**

```bash
systemctl stop clamd@scan && systemctl disable clamd@scan
systemctl stop puppet && systemctl disable puppet
systemctl stop salt-minion && systemctl disable salt-minion
systemctl stop chef-client && systemctl disable chef-client
```

**Back up the system** — critical, since some steps in the process are not reversible:

```bash
yum install -y sos
sosreport

tar --exclude=/proc --exclude=/sys --exclude=/dev --exclude=/run --exclude=/tmp \
  -czpvf /backup/centos-backup-$(date +%Y%m%d).tar.gz /
```

**Verify network access** to required Red Hat endpoints:

```bash
curl -I https://cdn.redhat.com
curl -I https://cdn-public.redhat.com
curl -I https://cert.console.redhat.com
curl -I https://subscription.rhsm.redhat.com
```

---

## Phase 2 — Update CentOS & Reboot (before installing convert2rhel)

CentOS mirrors are frequently stale/offline, so repos are redirected to the Vault archive first, then the system is fully updated and rebooted — this ensures the system is on the latest supported minor version and that rollback works correctly if conversion fails.

```bash
sed -i 's/^mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*
sed -i 's|#baseurl=http://mirror.centos.org|baseurl=https://vault.centos.org|g' /etc/yum.repos.d/CentOS-*

yum clean all
yum repolist

yum update -y
yum check-update
reboot
```

After reboot, confirm the kernel and repos are healthy before continuing:

```bash
uname -r
cat /etc/centos-release
yum repolist
```

---

## Phase 3 — Install convert2rhel

```bash
curl -o /etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release \
  https://security.access.redhat.com/data/fd431d51.txt

# Pick the repo file matching your TARGET RHEL version
curl -o /etc/yum.repos.d/convert2rhel.repo \
  https://cdn-public.redhat.com/content/public/repofiles/convert2rhel-for-rhel8-x86_64.repo

yum -y install convert2rhel
convert2rhel --version
```

### Configuring registration (two files must be set up before running the analysis)

**`/etc/convert2rhel.ini`** — controls conversion behavior and registration target:

```ini
[convert2rhel]
allow_unavailable_kmods = true
tainted_kernel_module_check_skip = true

[subscription_manager]
org = <your_org_name>
activation_key = <your_activation_key>
```

**`/etc/rhsm/rhsm.conf`** — points Subscription Manager at your registration source (Red Hat CDN or an internal Satellite server):

```ini
[server]
hostname = <your-satellite-or-cdn-hostname>
prefix = /rhsm
port = 443

[rhsm]
baseurl = https://<your-satellite-hostname>/pulp/content
manage_repos = 1
```

> If registering through an internal Satellite server, `hostname` and `baseurl` point to your Satellite instance instead of Red Hat's public CDN — Satellite uses a `/rhsm` prefix and serves content from its Pulp content store.

### Installing the Satellite CA certificate (Satellite-managed environments only)

This step is easy to miss — `/etc/rhsm/ca/` can be completely empty by default, which causes conversion to fail with `SSL: CERTIFICATE_VERIFY_FAILED` when it tries to register.

```bash
rpm -Uvh http://<your-satellite-hostname>/pub/katello-ca-consumer-latest.noarch.rpm

# If DNS can't resolve the hostname, add it to /etc/hosts first
echo '<satellite-ip> <your-satellite-hostname>' >> /etc/hosts

ls -lh /etc/rhsm/ca/
# Expected: katello-default-ca.pem, katello-server-ca.pem, redhat-uep.pem

openssl x509 -in /etc/rhsm/ca/katello-server-ca.pem -noout -dates

curl -v https://<your-satellite-hostname> \
  --cacert /etc/rhsm/ca/katello-server-ca.pem 2>&1 | grep -E 'SSL|certificate|HTTP|error'

subscription-manager register --org='<org>' --activationkey='<key>' --force

ls -lh /etc/pki/entitlement/
ls -lh /etc/pki/consumer/

subscription-manager refresh
subscription-manager auto-attach
```

**Checkpoint:** only proceed once the Satellite CA cert is installed, `/etc/rhsm/ca/` is populated, SSL is working, registration succeeds, and entitlement certs exist in both `/etc/pki/entitlement/` and `/etc/pki/consumer/`.

---

## Phase 4 — Configure RHEL Package Access

Three options, depending on environment:

- **Option A — Red Hat CDN via RHSM**: register directly against Red Hat using an activation key from the Hybrid Cloud Console (requires Simple Content Access enabled).
- **Option B — Red Hat Satellite**: register against an internal Satellite server. Requires the Content View assigned to the host to include the correct RHEL repositories (BaseOS + AppStream for RHEL 8/9), published and promoted to the right Lifecycle Environment.
- **Option C — Custom/local repos**: for air-gapped systems, point to a local RHEL package mirror instead.

```bash
subscription-manager repos --list | grep -i rhel
```

---

## Phase 5 — Pre-Conversion Analysis

Always run this dry-run before the real conversion — it makes no changes to the system.

```bash
convert2rhel analyze

# Custom repo (no RHSM) example
convert2rhel analyze --no-rhsm --enablerepo rhel-8-baseos --enablerepo rhel-8-appstream
```

Every check reports one of: **Success**, **Error** (must fix), **Overridable** (fix or bypass), **Warning**, **Skip**, or **Info**. Resolve all `Error`/`Overridable` items, then re-run the analysis until the report is clean.

---

## Phase 6 — Perform the Conversion

```bash
convert2rhel
# or, with custom repos:
convert2rhel --no-rhsm --enablerepo rhel-8-baseos --enablerepo rhel-8-appstream

# Watch progress in a separate terminal
tail -f /var/log/convert2rhel/convert2rhel.log
```

Partway through, convert2rhel shows a **point-of-no-return warning** — after confirming past it, there's no automatic rollback, so the Phase 1 backup is the only way back if something goes wrong.

If an SSH session drops mid-conversion, reconnect and check whether the process is still running rather than assuming it failed:

```bash
ps aux | grep convert2rhel
cat /var/run/lock/convert2rhel.pid
```

Once complete, reboot to load the new RHEL kernel:

```bash
reboot
cat /etc/redhat-release
uname -r
```

---

## Phase 7 — Post-Conversion Cleanup

```bash
yum remove -y convert2rhel
rm -f /etc/convert2rhel.ini /etc/convert2rhel.ini.rpmsave /etc/yum.repos.d/convert2rhel.repo

# Check for leftover packages with no RHEL equivalent
yum list extras --disablerepo='*' --enablerepo=rhel-7-server-rpms

# Re-enable anything disabled in Phase 1
systemctl enable clamd@scan && systemctl start clamd@scan
systemctl enable puppet && systemctl start puppet
```

Optionally, continue the upgrade path to the latest supported RHEL release using `leapp upgrade` (RHEL 7→8→9).

---

## Common Issues & Fixes

| Issue | Cause | Fix |
|---|---|---|
| `SSL: CERTIFICATE_VERIFY_FAILED` during registration | Satellite CA cert missing from `/etc/rhsm/ca/` | Reinstall the Katello CA consumer RPM from Satellite, verify the directory is populated |
| Registration fails silently | DNS can't resolve Satellite hostname | Add a temporary `/etc/hosts` entry for the Satellite server |
| Entitlement directories empty after registration | Subscription not auto-attached | Run `subscription-manager refresh && subscription-manager auto-attach` |
| Analysis reports package conflicts | Outdated or third-party packages with no RHEL equivalent | Update or remove flagged packages, re-run `convert2rhel analyze` |
| Conversion appears to hang after SSH disconnect | Session dropped, but process is still running server-side | Reconnect and check `ps aux \| grep convert2rhel` / tail the log — don't interrupt it |

---

## Lessons Learned

- The Satellite CA certificate step is the most commonly missed prerequisite in Satellite-managed environments — it fails silently as an SSL error that looks unrelated to certificates at first glance.
- Always run `convert2rhel analyze` and read the full report, even when no errors are shown — some issues only surface as warnings.
- The point-of-no-return warning is real — a verified backup before that point is the only safety net.
- SSH disconnects mid-conversion are recoverable — the process keeps running; reconnect and check the log instead of assuming failure.

---
*Documented from hands-on CentOS-to-RHEL conversion work in a Satellite-managed enterprise Red Hat environment.*
