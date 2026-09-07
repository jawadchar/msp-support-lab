# Lab day lookup

One page. Ctrl+F a word from the ticket.

---

## Read this before the call, not during

**Scope first, every ticket.** "Is this just this user, or others too?"

**Say the plan before the click.** "I want to confirm the current config before I change
anything, so I have a known state to go back to."

**One change at a time**, and say that you're doing it.

**Verify explicitly.** "That's applied. Let me confirm the original symptom is actually
gone rather than assume it."

**When looking something up.** "Let me double-check the exact order here — I want to
convert the mailbox before removing the license, not after."

**When stuck.** "I haven't found it yet. Here's what I've ruled out, here's what I'd
check next." — passing answer even if you never solve it. Silence is not.

**When the ticket is ambiguous.** "The ticket says add her to the security group but
doesn't say which — is there a naming convention?" Asking is scored.

**Close every ticket with a note.** What was asked, what you did, how you verified.
Two sentences. Most candidates skip this entirely and it is being watched.

---

## Do not do these

| | Why |
|---|---|
| **Static IP on an Azure VM's primary NIC** | Drops your own RDP session. Unrecoverable. Say "that's reserved at the Azure NIC level, not in the guest" |
| **Remove license before converting mailbox to shared** | Starts a 30-day deletion clock on the data |
| **Disable MFA to fix an MFA problem** | Use a Temporary Access Pass instead |
| **Delete a user when the ticket says disable** | Soft delete, 30-day clock, harder to undo |
| **Block sign-in and stop there** | Blocking ≠ revoking. Sessions live up to an hour |
| **Sign into a personal account on their VM** | Bad practice in front of people evaluating practice |

---

## First 60 seconds on any box

```
dsregcmd /status          # AzureAdJoined / DomainJoined / WorkplaceJoined, TenantName
whoami /groups            # what am I
ipconfig /all             # address, mask, gateway, DNS, DHCP or static
hostname
```

`dsregcmd` tells you what kind of environment you're in before you touch anything.

---

## M365 tickets

**Portals:** admin.cloud.microsoft (users, licenses, groups) · entra.microsoft.com
(auth methods, CA, sign-in logs) · admin.exchange.microsoft.com (mailboxes, DLs, mail
flow) · security.microsoft.com (quarantine) · intune.microsoft.com (devices)

### "forgot password" / "reset password"
Admin center → Users → Active users → select → **Reset password**. Leave
force-change-at-signin **checked**.
*Gotcha:* if hybrid, on-prem AD is the source of truth and this gets overwritten. Ask.
*Say:* "I'd verify the requester's identity out-of-band before resetting."

### "new phone" / "can't complete MFA" / "lost authenticator"
Entra → Users → select → **Authentication methods**. Issue a **Temporary Access Pass**.
*Never* disable MFA. TAP is time-boxed, single-use, auditable.
Fallback: "Require re-register multifactor authentication."

### "needs a license" / "wrong license" / "licensing"
Admin center → Users → select → **Licenses and apps**.
**First: see what the tenant actually owns.** Billing → Licenses. Usually only one or two
SKUs exist, which often decides it for you.
*Three questions that discriminate:* installed Office apps or web only? Device management
and security needed? Email only?
- Web apps only → **Business Basic**
- Installed apps, no device management → **Business Standard**
- Installed apps + Intune + Entra P1 + Defender → **Business Premium**
- Email and nothing else → **Exchange Online Plan 1** (P2 for 100 GB + archive)
- Over 300 seats → enterprise line. M365 E3 = P1 + Intune. M365 E5 = P2 + full Defender.

*Gotcha:* **usage location** is a regulatory field, not a billing address. The portal
auto-fills it from the tenant country; Graph/PowerShell fails without it. Watch for
multi-country clients silently inheriting the wrong one.
Service plans inside a license can be individually unchecked.
*Say:* "At scale I'd use group-based licensing rather than assigning per user."
*Say if unsure:* "I can see Standard and Premium available — she's a new hire on a
managed laptop so I'd lean Premium for the Intune coverage. Does that match?"

### "left the company" / "terminated" / "offboard"
Order: **block sign-in → reset password → revoke sessions → convert mailbox to shared →
reassign OneDrive owner → remove groups → remove license last.**
`Revoke-MgUserSignInSession -UserId x`
*Gotcha:* shared mailbox under 50 GB needs no license. Convert before delicensing.

### "shared mailbox" / "can't send from" / "can't send as"
Three independent permissions:
- **Full Access** → read only. `Add-MailboxPermission`
- **Send As** → appears as the mailbox. `Add-RecipientPermission`
- **Send on Behalf** → "Dana on behalf of Support". `Set-Mailbox -GrantSendOnBehalfTo`

Symptom "can read but can't send" = Full Access granted, Send As forgotten. ~90% of cases.
*Gotcha:* automapping is Outlook desktop only, Full Access only, may take hours.

### "convert to shared mailbox"
EAC → Recipients → Mailboxes → select → Others → **Convert to shared mailbox**.
**Convert first, remove license second.**

### "disable" / "block" / "suspend the account"
Admin center → Users → select → **Block sign-in**. Then **revoke sessions** separately.
Blocking stops new auth. Existing refresh tokens survive up to an hour.

### "add to group" / "needs access to"
| Type | Has email | Can grant permissions | Where |
|---|---|---|---|
| Security group | No | Yes | Admin center |
| Mail-enabled security | Yes | Yes | **Exchange only** |
| Microsoft 365 group | Yes | Yes | Admin center, makes SharePoint + Teams |
| Distribution list | Yes | **No** | Exchange |

If the ticket wants access **and** email, a plain security group is wrong.
*Gotcha:* dynamic groups can't be edited manually — fix the rule instead.

### "distribution list" / "email group" / "everyone at X should get"
EAC → Recipients → Groups → Add a group → **Distribution list**.
*Gotcha:* defaults to **internal senders only**. Allow external if outsiders must email it.
*Say:* "A DL fans out to individual inboxes. If they want shared history, a shared
mailbox fits better — should I confirm?"

### "not getting email from X"
**Message trace** first, always. EAC → Mail flow → Message trace.
Status `Delivered` → check `Get-InboxRule` and quarantine.
Status `Quarantined` → security.microsoft.com → Review → Quarantine → Release.
No result → never reached Microsoft. Sender's problem, or wrong address.

### "our email goes to their spam"
SPF `v=spf1 include:spf.protection.outlook.com -all` · DKIM = 2 CNAMEs
`selector1/2._domainkey` + enable in Defender · DMARC TXT at `_dmarc`.
Read `Authentication-Results` in a junked message header.
SPF fails + DKIM passes = **forwarding**. SPF breaks on forward, DKIM survives.
*Gotcha:* SPF has a 10-DNS-lookup limit; too many includes = permerror.

### "blocked" / "can't sign in" / access denied
Entra → **Sign-in logs** → find entry → **Conditional Access tab**. Names the exact
policy that failed. Then **What If** to model changes.
Codes: `50126` bad password · `50053` lockout/spray · `53000` not compliant ·
`53001` not joined · `53003` blocked by CA · `500121` MFA denied or timed out ·
`530035` requires app protection policy.
*Say:* "I'd deploy the fix in report-only first rather than excluding the user."

### "weird emails from my account" / suspected compromise
**Block → reset password → revoke sessions.** In that order.
Then persistence: `Get-InboxRule` · mailbox forwarding (`ForwardingSmtpAddress`) ·
**OAuth consent grants** (survive a password reset) · **attacker-added MFA methods**.
*Say:* "Modern AiTM kits steal the session cookie after MFA succeeds, so revocation is
mandatory — they hold a token, not a password."

### "restore deleted user"
Admin center → Users → **Deleted users** → Restore. 30-day window.

---

## Windows tickets

### "can't reach internal sites but internet works"
DNS pointing at a public resolver instead of internal. `ipconfig /all` → fix DNS.
**Control Panel → Network and Internet → Network and Sharing Center → Change adapter
settings → adapter → Properties → IPv4 → Properties.** (`ncpa.cpl` is the shortcut.)

### "add a DNS server"
Second one goes in **Alternate DNS**. Third or more: **Advanced → DNS tab → Add**.
*Say:* "Windows only falls to the alternate if the preferred one doesn't respond at all.
An NXDOMAIN is a valid answer, so a bad zone breaks resolution permanently and a
secondary server won't rescue it."

### "some sites work, others don't"
**Subnet mask.** Wrong mask = local range too narrow or too wide.
Gateway must be inside the subnet or Windows rejects it.

### "set a static IP"
**On an Azure VM: don't.** Say it's reserved at the NIC level in Azure; changing it in
the guest drops your RDP session.
Otherwise: IPv4 Properties → Use the following. Check it's **outside the DHCP scope**.
Verify: `ipconfig /all`, `ping gateway`, `nslookup name`, `nslookup name <server>`,
`ipconfig /flushdns` before retesting.

### "APIPA" / 169.254.x.x
No DHCP response. Scope disabled, scope exhausted, DHCP server unauthorized, or no link.

### "new disk isn't showing"
`diskmgmt.msc` → right-click disk number → **Initialize Disk** → **GPT**.
Then right-click unallocated → New Simple Volume → NTFS → quick format.
*If disk shows Offline:* right-click → Online. Server default SAN policy is Offline Shared.
*GPT vs MBR:* GPT >2TB, 128 partitions, backup partition table, needs UEFI to boot.

### "drive is full" / "extend the D drive"
`diskmgmt.msc` → right-click volume → Extend Volume.
**Greyed out unless unallocated space is immediately to the RIGHT, same disk.**
A recovery partition in between blocks it. Say so rather than clicking around.

### "shrink a partition"
Shrink offers far less than free space because of **unmovable files** — page file,
hibernation, shadow copies.

### "device not working"
`devmgmt.msc` → **View → Show hidden devices** (do this first, every time).
Yellow triangle → Properties → Device status → read the code.
`10` can't start · `28` no driver · `43` device reported a problem · `45` ghost/disconnected.

### "worked until the update"
Device Manager → device → Properties → **Driver tab → Roll Back Driver**.
Greyed out if there's no previous driver stored.

### "unknown device"
Properties → **Details tab → Hardware Ids** → `PCI\VEN_xxxx&DEV_xxxx`. Vendor and device
codes identify it exactly.

### "reinstall the device"
Uninstall device (leave "delete driver" unchecked unless the driver is the problem) →
**Action → Scan for hardware changes**.

---

## Docker

```
docker ps -a              # running? exited? restart-looping?
docker logs <name>        # why it died. read this before anything else
docker logs --tail 50 -f <name>
docker inspect <name> | grep -i -A5 state    # exit code
docker start <name>
docker restart <name>
```

| Symptom | Usually |
|---|---|
| Every command fails identically | Engine isn't running, not the container |
| Exited (0) | Process finished. Not an error — it has nothing to keep running |
| Exited (1) | App crashed. Read logs |
| Exited (137) | OOM killed, or SIGKILL |
| Exited (125/126/127) | Bad docker command, bad entrypoint, or command not found |
| Restarting loop | Crashes on start, restart policy retries. `docker logs` |
| "port is already allocated" | Something else on that port. `ss -tulpn \| grep <port>` |
| "no such file or directory" on start | Bad volume mount path or missing entrypoint |

```
docker compose ps
docker compose logs <service>
docker compose up -d
```

*Say:* "I'll check whether the container is stopped or crash-looping before I try to
start it, because those are different problems."
