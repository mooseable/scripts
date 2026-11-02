# scripts

This repository contains operations scripts for Linux servers.
Each script is in a folder for its subject.
Each script has a section in this file.

The text in this file and in the script comments is written in ASD-STE100 Simplified Technical English.

## Contents

| Folder | Script | Purpose |
| --- | --- | --- |
| [`cve-mitigations`](cve-mitigations/) | [`cve-2026-98432.sh`](cve-mitigations/cve-2026-98432.sh) | Check, mitigate and remove the mitigation for CVE-2026-98432 (Linux kernel, IPv4 TCP). |

## General rules for all scripts

1. Read a script before you run it on a server.
2. Run the scripts as root, unless the script section gives a different instruction.
3. Test a script on one server before you use it on many servers.
4. Each script shows its commands when you run it without arguments.

---

## CVE-2026-98432 (`cve-mitigations/cve-2026-98432.sh`)

### Summary

CVE-2026-98432 is a use-after-free fault in the Linux kernel.
The fault is in the code that receives IPv4 TCP connection requests (SYN packets).
A remote attacker can send a SYN packet with IPv4 header options to a listening TCP port.
This packet can cause the fault. No credentials are necessary.

- CVSS 3.1 score from the kernel CVE team: 9.8 (Critical).
- The fault is in Linux 4.4 and later. RHEL 7 and CentOS 7 are not affected.
- IPv6 is not affected by the main fault.
- The published root exploit used an MPTCP listener. The fault is in plain IPv4 TCP. Thus, a host without MPTCP can also be vulnerable.
- The trigger is **IPv4 header options** (IHL > 5). The trigger is not TCP options. All usual SYN packets have TCP options. Usual traffic does not have IPv4 header options.

### Supported systems

- Ubuntu 20.04 and later.
- Debian 12 and 13.
- AlmaLinux, Rocky Linux and RHEL 8, 9 and 10.

On other systems, the script does only the general checks.

### Requirements

- `bash`, `python3` and `nft` (the `nftables` package).
- `ss` (`iproute2`) to find MPTCP listeners. Version 5.10 or later is necessary for `ss -M`.
- `systemd` for persistence.
- Root access.

### Run the check in one command

This command gets the script and runs the check. It does not keep the file and does not make changes.

```bash
curl -fsSL https://raw.githubusercontent.com/mooseable/scripts/main/cve-mitigations/cve-2026-98432.sh | sudo bash -s check
```

If `curl` is not installed, use `wget`:

```bash
wget -qO- https://raw.githubusercontent.com/mooseable/scripts/main/cve-mitigations/cve-2026-98432.sh | sudo bash -s check
```

The exit code of the command is the result. Refer to [Exit codes](#exit-codes).

> [!NOTE]
> The `mitigate` command cannot run through a pipe. It installs a copy of the script for the systemd unit. Thus, it needs a file. Use the next procedure.

### Get the script, keep it, and run the check

```bash
curl -fsSL -o cve-2026-98432.sh https://raw.githubusercontent.com/mooseable/scripts/main/cve-mitigations/cve-2026-98432.sh && chmod +x cve-2026-98432.sh && sudo ./cve-2026-98432.sh check
```

### Check many servers

This command runs the check on each server in `hosts.txt` through SSH. It shows one result line for each server.

```bash
while read -r h; do
  curl -fsSL https://raw.githubusercontent.com/mooseable/scripts/main/cve-mitigations/cve-2026-98432.sh | ssh "$h" 'sudo bash -s check' >/dev/null 2>&1
  echo "$h exit=$?"
done < hosts.txt
```

### Commands

| Command | Changes the host | Description |
| --- | --- | --- |
| `check` | No | Show the kernel status, the fixed package status, the mitigation status and the exposure. |
| `precheck [seconds]` | Temporary only | Count the incoming TCP packets with IPv4 options for a sample time (default 60, minimum 5). The packets are not dropped. Show the MPTCP listeners and the Go programs that listen on TCP. |
| `mitigate` | Yes | Add the nftables drop rule, make it persistent, and set MPTCP to off. |
| `ensure` | Yes, if necessary | Apply the drop rule again if it is missing or changed. The systemd timer runs this command. |
| `unmitigate` | Yes | Remove the mitigation. Set MPTCP to the value that it had before `mitigate`. |
| `version <release>` | No | Show the upstream result for a kernel release string, for example `6.12.113`. Use it with a list of kernel versions from an inventory. |

### Exit codes

| Code | Meaning for `check` |
| --- | --- |
| 0 | Fixed. The running kernel or a live patch contains the fix. |
| 1 | Vulnerable. There is no correct drop rule. |
| 2 | Vulnerable, but mitigated. The drop rule is live and correct. |
| 3 | Not known. For example, the script is not root, or `nft` or `python3` is missing. Treat the host as vulnerable. |

For `mitigate`, `ensure` and `unmitigate`: 0 means that all steps are successful. 1 means that one or more steps failed.

### Procedure: mitigate a vulnerable host

1. Run the check:
   ```bash
   sudo ./cve-2026-98432.sh check
   ```
2. If the result is 0, stop. The host is fixed.
3. If a fixed kernel is available, install it and reboot. Then do step 1 again.
4. If no fixed kernel is available, run the precheck for 5 minutes or more:
   ```bash
   sudo ./cve-2026-98432.sh precheck 300
   ```
5. If the precheck shows packets with IP options, find their sources before you continue.
6. Apply the mitigation:
   ```bash
   sudo ./cve-2026-98432.sh mitigate
   ```
7. Make sure that the result shows three `OK` lines (live protection, persistence, MPTCP off) and that the exit code is 0.
8. Restart the services that `mitigate` shows as MPTCP listeners. These listeners stay open until the services restart.
9. Monitor the dropped packets for one week:
   ```bash
   journalctl -k | grep CVE-2026-98432-drop
   ```
10. When a fixed kernel is available, install it and reboot. Then remove the mitigation:
    ```bash
    sudo /usr/local/sbin/cve-2026-98432.sh unmitigate
    ```

### What `mitigate` changes

| Item | Path or setting |
| --- | --- |
| nftables table | `table ip cve_2026_98432` (drop rule for IPv4 TCP with IP options) |
| Rule file | `/etc/cve-2026-98432.nft` |
| Script copy for systemd | `/usr/local/sbin/cve-2026-98432.sh` |
| systemd unit and timer | `/etc/systemd/system/cve-2026-98432-mitigation.service` and `.timer` |
| MPTCP setting | `net.mptcp.enabled = 0` in `/etc/sysctl.d/90-cve-2026-98432-disable-mptcp.conf` |
| State for rollback | `/var/lib/cve-2026-98432/` |

The unit applies the rule at boot, before `network-pre.target`.
The timer examines the rule every 2 minutes. If a firewall reload removes or changes the rule, the timer applies it again.
`unmitigate` removes all of these items. It restores the MPTCP value and the sysctl file that were present before `mitigate`.

### Hook selection

The default hook is `prerouting` with priority -300. This hook operates before DNAT. Thus, it also protects containers that receive traffic through published ports.

The `prerouting` hook also drops forwarded TCP packets with IP options. If the host is a router and has no containers, use the `input` hook:

```bash
sudo MITIGATE_HOOK=input ./cve-2026-98432.sh mitigate
```

### Limits

- The mitigation is temporary. It is not a fix. Install a fixed kernel as soon as possible.
- The rule protects the host network namespace and the containers that receive traffic through it. Containers and VMs with their own routed IP addresses (macvlan, SR-IOV, bridged VMs) need their own rule or kernel.
- Do not apply the rule on hosts that use CIPSO or NetLabel. CIPSO uses IP options.
- After a firewall reload that flushes the rule set, the host is not protected until the next timer run (2 minutes or less).
- On Ubuntu and EL, the script uses the vendor package changelog of the running kernel to find the fix. The kernel version number does not show backported fixes.

### Recommended network edge rule

For threats from external networks, also apply the same rule on the edge firewall or load balancer. One edge rule protects all servers behind it.
The rule: drop IPv4 TCP packets with IP header options (IHL > 5).

### Fix status (2026-10-30)

| Distribution | Status |
| --- | --- |
| Upstream stable | Fixed in 5.10.272, 5.15.223, 6.1.190, 6.6.159, 6.12.113, 6.18.56, 7.2.10 and 7.3. |
| Debian 12 and 13 | Fixed. |
| AlmaLinux 8, 9 and 10 | Fixed kernels are in the testing repository. |
| Rocky Linux and RHEL 8, 9 and 10 | No fix. No release date. |
| Ubuntu 22.04, 24.04, 26.04 and 26.10 | Fix built, not published ("pending"). |
| Ubuntu 16.04, 18.04 and 20.04 | No fix. |

This status changes frequently. The `check` command gets the current Ubuntu and EL status when it runs.

### References

- [Linux kernel CVE announcement](https://git.kernel.org/pub/scm/linux/security/vulns.git/plain/cve/published/2026/CVE-2026-98432.mbox)
- [Upstream fix e3f33b0a1d89](https://git.kernel.org/stable/c/e3f33b0a1d89d738e2b913dab519226eaf76e15b)
- [Ubuntu CVE tracker](https://ubuntu.com/security/CVE-2026-98432)
- [Red Hat CVE page](https://access.redhat.com/security/cve/cve-2026-98432)
- [Debian security tracker](https://security-tracker.debian.org/tracker/CVE-2026-98432)
- [AlmaLinux announcement](https://almalinux.org/blog/2026-10-30-cve-2026-98432/)

---

## Add a script to this repository

1. Put the script in a folder for its subject. For example, put CVE mitigations in `cve-mitigations/`.
2. Add a row to the [Contents](#contents) table.
3. Add a section to this file. Use the same structure as the CVE-2026-98432 section: summary, supported systems, one-command run, commands, exit codes, what the script changes, limits and references.
