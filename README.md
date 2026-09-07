# MSP support reference

Working reference material for Microsoft 365 administration and Windows endpoint
support, plus a self-contained Hyper-V lab for practicing break-fix.

Written for a small-MSP context, where one person covers identity, mail, endpoints and
whatever else the ticket queue produces.

## Contents

| | |
|---|---|
| [`docs/m365-admin-tasks.md`](docs/m365-admin-tasks.md) | Everyday M365 admin tasks — passwords, licensing, mailbox conversion, groups, distribution lists. Portal path, PowerShell equivalent, and the failure mode for each |
| [`docs/m365-scenarios.md`](docs/m365-scenarios.md) | Nine diagnostic scenarios — mail delivery, MFA lockouts, Conditional Access, SharePoint access, account compromise. Ordered diagnostic paths with causes ranked by real frequency |
| [`docs/windows-gui-drills.md`](docs/windows-gui-drills.md) | Disk Management, Device Manager and network configuration, with seventeen hands-on drills that run on virtual disks so nothing real is at risk |
| [`docs/practice-tickets.md`](docs/practice-tickets.md) | Six tickets in realistic end-user phrasing, with a worked key. Three of them fail if executed literally |
| [`docs/lab-day-lookup.md`](docs/lab-day-lookup.md) | Single-page quick reference, keyed on ticket phrasing rather than topic |
| [`docs/lab-assistant-prompt.md`](docs/lab-assistant-prompt.md) | Context prompt for using an LLM as a troubleshooting assistant, with output constraints that keep it usable under time pressure |
| [`lab/`](lab/) | PowerShell for building a two-VM Hyper-V AD lab and injecting faults into it at random |

## The break-fix lab

`lab/New-LabVMs.ps1` builds a domain controller and a domain-joined client on an isolated
internal switch. `lab/Initialize-Lab.ps1` populates AD with OUs, users, groups, a DHCP
scope, a file share and a GPO.

`lab/Invoke-LabFault.ps1` then breaks one thing at random and logs what it did to a file
you're not supposed to read until you've diagnosed it. Six faults: DHCP scope
deactivated, client DNS misdirected, share and NTFS permissions set against each other,
account locked out from the client, GPO link moved to the wrong OU, and the computer
account password reset to break the secure channel.

The randomness is the point. Breaking something yourself rehearses the fix but not the
diagnosis, and diagnosis is the harder half. `lab/Reset-Lab.ps1` reverts both VMs to a
clean checkpoint in about thirty seconds, which is what makes repetition practical.

Requires Windows 10/11 Pro with Hyper-V, 16 GB RAM, and evaluation ISOs for Windows
Server and Windows 11 Enterprise.

## A few things in here worth the click

**DNS servers do not fail over the way most people think.** Windows only falls to the
alternate if the preferred server doesn't respond at all. NXDOMAIN is a valid response,
so a server that's up with a broken zone breaks resolution permanently and adding a
secondary doesn't help.

**Blocking sign-in is not revoking sessions.** Existing refresh tokens survive a block
and a password reset. In compromise response the order matters: reset, then revoke.

**Extend Volume needs adjacent space.** Unallocated space immediately to the right, on
the same physical disk. A recovery partition in between makes "just make D: bigger"
impossible with the built-in tool.

**Convert before delicensing.** A user mailbox needs a license to exist; a shared mailbox
doesn't. Removing the license first disconnects the mailbox and starts a 30-day clock.

## Notes

Everything here is written against a lab tenant. Nothing contains credentials or client
data. The default lab password lives in the script parameters and the environment has no
route to any real network — don't adapt these scripts to one without changing that first.
