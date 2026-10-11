# scanlogd for Enterprise Linux 7

Unofficial RPM packaging of [Openwall scanlogd](https://www.openwall.com/scanlogd/) 2.2.8 for **Red Hat Enterprise Linux 7** and **CentOS Linux 7**, using a native **systemd** service instead of the original SysV init script.

This repository is an independent packaging project and is **not an official Openwall release**. The EL7 build and runtime status must be established independently; successful tests on EL9 do not establish EL7 compatibility.

> **End-of-life warning:** CentOS Linux 7 reached end of life on June 30, 2024, and no longer receives security updates. RHEL 7 has also passed its standard maintenance period; any extended support depends on the applicable Red Hat subscription and lifecycle. This repository is intended for compatibility testing, migration, or explicitly supported legacy installations. Building an RPM does **not** make an unsupported operating system secure.

## Overview

scanlogd detects certain TCP port scans and writes a record to syslog. On Linux, its native input method uses a TCP raw socket and does not require libpcap. The daemon **does not block traffic by itself**. Optional Fail2Ban integration can use scanlogd's logs to block source IP addresses through iptables.

It is not a replacement for a default-deny firewall, a WAF, an IDS/IPS, operating-system security updates, or monitoring.

## Packaging changes compared with upstream

The original Openwall RPM specification installs a SysV init script and uses `chkconfig`. The intended EL7 adaptation:

- Installs a native `scanlogd.service` in the systemd unit directory.
- Uses EL7-compatible RPM systemd installation/removal scriptlets.
- Builds the upstream Linux raw-socket implementation, without libpcap.
- Preserves scanlogd's internal startup privilege drop to a dedicated `scanlogd` account.
- Provides a root-owned, empty chroot directory; the path compiled into `params.h` must match the directory installed by the SPEC.
- Includes the upstream manual, documentation, and license.

**EL7 compatibility notes:** The RPM and systemd versions in EL7 are older than in EL8-EL10. Confirm the SPEC does not depend on newer macros such as `%make_build`, `%{build_ldflags}`, or an unavailable `systemd-rpm-macros` package. Check that the required systemd RPM macros exist *inside the EL7 Mock build environment*. systemd 219 may not recognize every hardening directive accepted by newer systemd releases; inspect unit verification output and test the service on EL7.

Do **not** set `User=scanlogd` in the unit without changing and revalidating the daemon's startup sequence: scanlogd must first open its raw socket and then drops privileges internally.

## Repository layout

Recommended layout:

```text
scanlogd-el7/
├── README.md
├── SPECS/
│   └── scanlogd.spec
└── SOURCES/
    └── scanlogd.service
```

`RPMS/`, `SRPMS/`, and `Logs/` may be used for build artifacts. The upstream `scanlogd-2.2.8.tar.gz` file should be downloaded from Openwall rather than modified without attribution.

If your repository stores the SPEC or service unit at different paths, adjust the commands below. The source RPM must contain all files referenced by its SPEC.

## Build an EL7 RPM with Mock

An EL8/EL9 host can be used to run Mock, provided a working EL7 configuration and access to archived EL7 repositories are available. Run these commands from the build host, not from an untrusted production server.

### 1. Install build tools on the host

On an AlmaLinux 9 build host with the appropriate repositories configured:

```bash
dnf install -y mock mock-core-configs rpm-build curl git
```

Use root or a properly configured member of the `mock` group. Then check the installed configs:

```bash
ls /etc/mock/*7*x86_64*.cfg
```

Some Mock installations provide `epel-7-x86_64.cfg`; others no longer include an EL7 template. This README uses **`epel-7-x86_64` only as an example**, not as a guaranteed available configuration. If you create a custom archived EL7 config, substitute its actual name in each command.

### 2. Configure archived EL7 repositories

CentOS Linux 7 and EPEL 7 are archived. Old `mirrorlist=` entries may no longer work. An EL7 Mock config must point to working, appropriately verified repositories. Useful archive roots include:

```text
https://vault.centos.org/7.9.2009/os/x86_64/
https://vault.centos.org/7.9.2009/updates/x86_64/
https://vault.centos.org/7.9.2009/extras/x86_64/
https://archives.fedoraproject.org/pub/archive/epel/7/x86_64/
```

The exact EPEL archive path, metadata availability, TLS support, and repository configuration format must be checked for your Mock version. On RHEL 7, use repositories you are authorized to access instead of assuming CentOS binaries are interchangeable in every support scenario. Preserve repository GPG verification; do not disable signature checks simply to make an archived build succeed.

Test the Mock environment before attempting the RPM build:

```bash
mock -r epel-7-x86_64 --init
```

If initialization fails, fix the archive configuration and repository metadata first. A valid EL7 Mock configuration is a prerequisite for reproducible EL7 builds.

### 3. Get the packaging files and upstream source

```bash
git clone https://github.com/redadmin-k/scanlogd-el7.git
cd scanlogd-el7
mkdir -p build/SOURCES build/SRPMS build/RPMS
curl -fL https://download.openwall.net/pub/projects/scanlogd/scanlogd-2.2.8.tar.gz -o build/SOURCES/scanlogd-2.2.8.tar.gz
```

If `SPECS/scanlogd.spec` contains `Source1: scanlogd.service`, place the actual unit file at the expected path:

```bash
cp SOURCES/scanlogd.service build/SOURCES/scanlogd.service
```

Do not run the `cp` command if the repository uses a different service-unit location; use the unit file referenced by the SPEC. Check the inputs:

```bash
grep -E '^(Name|Version|Release|Source[0-9]*|BuildRequires):' SPECS/scanlogd.spec
ls -lh build/SOURCES/
```

The official source archive also has a detached signature. Verify the source against a trusted Openwall signing key before packaging for distribution:

<https://download.openwall.net/pub/projects/scanlogd/scanlogd-2.2.8.tar.gz.sign>

### 4. Verify RPM macros and dependencies

Before compiling, ensure the **EL7 SPEC**, rather than the EL9 SPEC, is checked in. In particular, EL7 packaging may require simpler build commands (such as `make` in `%build`) and different BuildRequires than newer distributions.

After successful Mock initialization, inspect the macro definitions in the EL7 environment where available:

```bash
mock -r epel-7-x86_64 --chroot 'rpm --eval "%{?dist} %{_unitdir} %{?systemd_post:systemd_post_available}"'
```

The release tag should resolve to `.el7` for a conventional EL7 configuration. If it does not, inspect the Mock config and SRPM metadata before publishing a build.

### 5. Build the EL7 source RPM

Use Mock's `--buildsrpm` so the distribution macro is evaluated in the EL7 build environment, even when the build host itself runs EL9:

```bash
mock -r epel-7-x86_64 --resultdir "$PWD/build/SRPMS" --buildsrpm --spec "$PWD/SPECS/scanlogd.spec" --sources "$PWD/build/SOURCES"
```

If the SPEC uses `Version: 2.2.8` and `Release: 1%{?dist}`, the expected source RPM is:

```text
build/SRPMS/scanlogd-2.2.8-1.el7.src.rpm
```

The actual filename is determined by the SPEC's `Release` field. Confirm it with `ls` rather than assuming the same release on every build.

### 6. Build the binary RPM

```bash
mock -r epel-7-x86_64 --resultdir "$PWD/build/RPMS" --rebuild "$PWD/build/SRPMS/scanlogd-2.2.8-1.el7.src.rpm"
```

Inspect the outputs:

```bash
ls -lh build/RPMS/
rpm -qpi build/RPMS/scanlogd-*.x86_64.rpm
rpm -qpl build/RPMS/scanlogd-*.x86_64.rpm
```

For failures, read `build/RPMS/build.log` and `build/RPMS/root.log`. A successful SRPM build alone does not prove that the source compiles for EL7.

## Install and test on an EL7 host

Perform runtime testing on a separate, authorized RHEL 7 or CentOS 7 test system or container, with networking and capabilities configured correctly.

On a test system with a compatible installed RPM, run:

```bash
yum localinstall -y ./scanlogd-2.2.8-1.el7.x86_64.rpm
systemctl enable --now scanlogd
systemctl status scanlogd --no-pager -l
pgrep -a scanlogd
systemctl cat scanlogd
```

Check that the systemd unit is compatible with the systemd version shipped by EL7:

```bash
systemd-analyze verify /usr/lib/systemd/system/scanlogd.service
journalctl -u scanlogd -b --no-pager
```

Some EL7 systemd versions may ignore unsupported unit settings. Review warnings and fix compatibility issues rather than assuming that `active (running)` proves full service hardening.

Check the logs:

```bash
journalctl -t scanlogd -f
grep 'scanlogd\[' /var/log/messages | tail -20
```

The `/var/log/messages` check requires syslog configuration that writes the scanlogd event there.

## Reproduce TCP port-scan detection

Only scan systems you own or have explicit permission to test. Use a separate external scanner whose IP is not required for management access. When Fail2Ban is enabled, a successful test may intentionally block the scanning IP.

From the external test host:

```bash
sudo nmap -n -Pn -sS -T4 --max-retries 0 -p 22,25,53,80,110,143,443,465,587,993,995 <authorized-test-host>
```

On the target system:

```bash
tail -f /var/log/messages | grep --line-buffered 'scanlogd\['
```

Illustrative event format (**not an EL7 test result**):

```text
Oct 11 10:41:17 server scanlogd[12345]: 203.0.113.35:39864 to 192.0.2.50 ports 110, 143, 80, 22, 25, 53, ...
```

scanlogd uses a destination-port count, weighted threshold, and timing rules. A short scan against too few reachable ports may not be detected. TCP packets discarded by upstream or local INPUT filtering may never reach scanlogd's raw socket. In an LXC environment, it observes only traffic delivered into the container's network namespace.

To distinguish logging failures from network limitations, inspect the service, socket, syslog configuration, and firewall rules. If needed, use `tcpdump` on a controlled test host to confirm SYN arrival.

## Optional: Fail2Ban with iptables

The scanlogd RPM alone does **not** block IPs. The following example assumes a separately installed, functional Fail2Ban package and a syslog file at `/var/log/messages`.

Create `/etc/fail2ban/filter.d/scanlogd.conf`:

```ini
[Definition]
failregex = ^.*scanlogd\[\d+\]:\s+<HOST>:\d+\s+to\s+\S+\s+ports\b
ignoreregex =
```

Create `/etc/fail2ban/jail.d/scanlogd.local`:

```ini
[scanlogd]
enabled = true
filter = scanlogd
backend = auto
logpath = /var/log/messages
maxretry = 1
findtime = 60
bantime = 600
banaction = iptables-allports
```

**The syntax and available actions depend on the installed Fail2Ban version.** Verify that `iptables-allports` exists before relying on it. The example is for controlled tests and should not be enabled blindly on a production server. Set appropriate `ignoreip` exclusions for critical administrative and monitoring hosts.

Verify the filter, configuration, and actions:

```bash
fail2ban-regex /var/log/messages /etc/fail2ban/filter.d/scanlogd.conf --print-all-matched
fail2ban-client -t
systemctl restart fail2ban
fail2ban-client status scanlogd
```

Then repeat the authorized external scan and confirm both the ban and the firewall rule:

```bash
fail2ban-client status scanlogd
iptables -S | grep f2b-scanlogd
iptables -nvL f2b-scanlogd --line-numbers
```

The actual chain name depends on the Fail2Ban action. A source IP in `Banned IP list` proves a Fail2Ban decision, but actual packet filtering must also be verified in iptables, including the chain jump and packet counters.

### Security limitations

- **IP spoofing:** scanlogd does not require a completed TCP connection. A forged source IP can cause an innocent address to be banned, so permanent automatic bans based only on scanlogd output are unsafe.
- **Incomplete coverage:** slow, distributed, or narrowly targeted scans can evade detection.
- **Existing firewall rules:** dropped packets may be invisible to scanlogd and prevent its detection threshold from being reached.
- **False positives:** automated all-port blocks can interrupt legitimate users or administration.
- **Resource cost:** check logging volume, CPU, memory, and firewall rule count on busy services.

## Verification status

As of **2026-10-11**, EL7-specific Mock build and runtime validation have **not been provided** for this repository. Replace the entries below only after collecting actual results from an EL7 build and test system.

| Test | EL7 status |
| --- | --- |
| Mock source RPM build | Not yet documented |
| Mock binary RPM build | Not yet documented |
| Native systemd service startup | Not yet documented |
| TCP raw-socket operation | Not yet documented |
| External TCP scan detection | Not yet documented |
| Fail2Ban automatic ban | Not yet documented |
| iptables packet drop | Not yet documented |
| SELinux enforcing mode | Not yet documented |
| LXC operation | Not yet documented |

A separate AlmaLinux 9 test confirmed detection, automatic Fail2Ban banning, and an iptables DROP counter increment. Those are **EL9 results, not EL7 results**.

## Troubleshooting

- `No such config file`: install or create an EL7 Mock configuration and use its exact name with `-r`.
- Failed Yum/Mock repository downloads: EL7 is archived; switch the Mock config to verified Vault/EPEL archive URLs.
- `Bad file: scanlogd.service`: put the service unit in `build/SOURCES` and check the `Source1` path.
- Unknown RPM macro or package: adapt the SPEC to EL7's RPM/systemd versions. Do not assume EL9 macros or dependencies exist on EL7.
- Binary RPM built but service fails: inspect `journalctl -u scanlogd`, unit compatibility, capabilities, and chroot directory permissions.
- No scan records: confirm packet reachability, detection threshold, logger output, and iptables rules.
- Scan records but no Fail2Ban action: check `fail2ban-regex`, `ignoreip`, and Fail2Ban logs.
- Fail2Ban bans but no actual DROP: inspect iptables jump ordering and per-chain packet counters.
- SELinux denial: inspect AVC logs; do not disable SELinux as a substitute for understanding the problem.

## Upstream source and license

- Openwall project: <https://www.openwall.com/scanlogd/>
- Upstream source repository: <https://github.com/openwall/scanlogd>
- Openwall source archive: <https://download.openwall.net/pub/projects/scanlogd/>
- Upstream license: <https://github.com/openwall/scanlogd/blob/main/LICENSE>
- CentOS 7 end-of-life information: <https://blog.centos.org/2023/04/end-dates-are-coming-for-centos-stream-8-and-centos-linux-7/>

scanlogd is distributed under a simplified BSD-like license. The upstream SPEC calls it `BSD-compatible`; this is not necessarily identical to standard SPDX `BSD-2-Clause` or `BSD-3-Clause` terms. Keep the original LICENSE with redistribution and review the exact wording before publishing license metadata.

This project provides **unofficial EL7 RPM packaging** and does not claim that Openwall, Red Hat, or CentOS endorses this repository.

