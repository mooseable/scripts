# scripts

This repository contains operations scripts for servers and network devices.
Each script is in a folder for its subject.
Each mitigation or script has its own document. The index below links to each document.

The text in this repository is written in ASD-STE100 Simplified Technical English.

## Index

### CVE mitigations (`cve-mitigations/`)

| CVE | Description | Platforms | Document |
| --- | --- | --- | --- |
| CVE-2026-98432 | Linux kernel IPv4 TCP use-after-free (SYN with IPv4 options) | Linux (Ubuntu, Debian, AlmaLinux, Rocky Linux, RHEL), MikroTik RouterOS | [cve-2026-98432.md](cve-mitigations/cve-2026-98432.md) |

## General rules

1. Read a script before you run it.
2. Test a script on one host or device before you use it on many.
3. Run the Linux scripts as root, unless the document gives a different instruction.
4. Use the check or audit step before you apply a change.
5. Each Linux script shows its commands when you run it without arguments.

## Get a script

All files are available from this address:

```
https://raw.githubusercontent.com/mooseable/scripts/main/<folder>/<file>
```

Run a Linux script directly (for read-only commands, for example `check`):

```bash
curl -fsSL https://raw.githubusercontent.com/mooseable/scripts/main/<folder>/<file> | sudo bash -s <command>
```

Get a Linux script and keep it:

```bash
curl -fsSL -o <file> https://raw.githubusercontent.com/mooseable/scripts/main/<folder>/<file> && chmod +x <file>
```

Get and run a MikroTik script (RouterOS terminal):

```
/tool fetch url="https://raw.githubusercontent.com/mooseable/scripts/main/<folder>/<file>.rsc" dst-path="<file>.rsc"; :delay 1s; /import file-name="<file>.rsc"
```

Each document gives the full commands for its scripts.

## Add a script or mitigation

1. Put the files in a folder for their subject. For example, put CVE mitigations in `cve-mitigations/`.
2. Give each file a name that starts with the CVE number or the subject. For example: `cve-2026-98432.sh`, `cve-2026-98432-mikrotik-apply.rsc`.
3. Write one document for the mitigation, in the same folder: `<name>.md`.
4. Add a row to the [Index](#index).
5. Use this structure for a mitigation document:
   1. Summary
   2. Files
   3. Use the script (one-command check, get and keep, commands, exit codes, procedure, changes)
   4. Manual steps (for each platform)
   5. Special cases (for example containers and virtual machines)
   6. Test the mitigation
   7. Remove the mitigation
   8. Limits
   9. Questions
   10. Fix status (with the date)
   11. References
6. Add a link to the document in the header of each script.
7. Write all text in ASD-STE100 Simplified Technical English.
