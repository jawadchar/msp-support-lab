# M365 admin task runbook

The tasks named in the interview brief, each with the portal path, the PowerShell
equivalent, and the specific thing that goes wrong. These are execution tasks — you are
being scored on doing them cleanly and quickly, not on diagnosing them.

Drill each one until you can do it without hunting for the blade.

**Portals, and knowing which is which:**

| Portal | URL | What lives here |
|---|---|---|
| Microsoft 365 admin center | admin.cloud.microsoft | Users, licenses, groups, most daily tasks |
| Entra admin center | entra.microsoft.com | Identity depth: auth methods, CA, sign-in logs, roles |
| Exchange admin center | admin.exchange.microsoft.com | Mailboxes, distribution lists, mail flow, message trace |
| Defender | security.microsoft.com | Quarantine, threat policies, alerts |
| Purview | purview.microsoft.com | Audit log, retention, DLP |
| Intune | intune.microsoft.com | Devices, compliance, app deployment |

Fumbling between portals is the most visible way to look unfamiliar. Most of what's
described below is in the M365 admin center; mailbox and distribution list work is in
Exchange.

---

## Prerequisites

### Portals work fine from Linux

Every portal above is browser-only. Nothing in this runbook requires Windows to click
through, and the portals are where most of the drilling happens anyway.

### PowerShell from Arch

Both modules run on PowerShell 7 on Linux. Install from the AUR, or use the tarball
release from Microsoft:

```bash
yay -S powershell-bin      # or: paru -S powershell-bin
pwsh
```

Then inside `pwsh`. **Install the Graph submodules you need, not the full meta-module,**
and keep every `Microsoft.Graph.*` module on the same version:

```powershell
Install-Module Microsoft.Graph.Authentication -Scope CurrentUser -Force
Install-Module Microsoft.Graph.Users -Scope CurrentUser -Force
Install-Module Microsoft.Graph.Groups -Scope CurrentUser -Force
Install-Module Microsoft.Graph.Identity.DirectoryManagement -Scope CurrentUser -Force

Install-Module ExchangeOnlineManagement -Scope CurrentUser -Force
```

### Never load Exchange and Graph in the same session

This is the one that will bite you. Both modules ship their own copy of
`Microsoft.Identity.Client` (MSAL), and .NET only loads one version of an assembly per
process. Whichever module imports first wins, and the second one then calls methods that
don't exist in the version already loaded. The symptom is:

```
Connect-MgGraph: DeviceCodeCredential authentication failed: Method not found:
'!0 Microsoft.Identity.Client.BaseAbstractApplicationBuilder`1.WithLogging(...)'
```

That's not an auth failure despite what it says. It's a binary compatibility error
surfacing through the auth path.

**Fix: run them in separate `pwsh` sessions.** Two terminal tabs, Exchange in one, Graph
in the other. There is no configuration that makes them coexist reliably.

The same conflict happens with the `Az` modules, which also bundle MSAL. If `Az.Accounts`
is loaded, Graph will break the same way.

### If it's already broken

Mixed `Microsoft.Graph.*` versions cause this even in a clean session. Check first:

```powershell
Get-Module Microsoft.Graph* -ListAvailable | Select-Object Name, Version | Sort-Object Name
```

Every row should show the same version. If they don't, wipe and reinstall:

```powershell
Get-InstalledModule Microsoft.Graph* | ForEach-Object {
    Uninstall-Module $_.Name -AllVersions -Force -ErrorAction SilentlyContinue
}
```

`Uninstall-Module` leaves directories behind more often than it should on Linux. Confirm
and clear manually:

```bash
ls ~/.local/share/powershell/Modules | grep Microsoft.Graph
rm -rf ~/.local/share/powershell/Modules/Microsoft.Graph*
```

Then reinstall the four submodules above in a **fresh** `pwsh` session.

### Connecting

```powershell
# session one
Connect-ExchangeOnline -UserPrincipalName Jawad@SprawlDefense.onmicrosoft.com -Device

# session two, separate terminal
Connect-MgGraph -Scopes 'User.ReadWrite.All','Group.ReadWrite.All','Organization.Read.All' -UseDeviceCode
```

**Device code auth is mandatory on Linux.** There's no WAM broker and no interactive
browser popup, so both modules need the device code flow — you get a code, open
microsoft.com/devicelogin in a browser, and paste it. `Connect-ExchangeOnline` uses
`-Device`, `Connect-MgGraph` uses `-UseDeviceCode`. Different flag names for the same
thing, which is exactly the kind of small friction that eats ten minutes if you don't
know it in advance.

**If device code is blocked by Conditional Access**, you'll get AADSTS530035 or
AADSTS53003 rather than a module error. That's a policy problem, not a PowerShell
problem — check Entra sign-in logs, Conditional Access tab, and find the policy by name.

### Seed test users first

You need bodies to operate on. Every task below needs a target that isn't your admin
account, and shared mailbox permissions specifically need two people. You have 25 E5
seats, so use six.

```powershell
$domain = 'SprawlDefense.onmicrosoft.com'
$e5 = (Get-MgSubscribedSku | Where-Object SkuPartNumber -eq 'SPE_E5').SkuId
$pw  = @{ forceChangePasswordNextSignIn = $false; password = 'LabPass!2026x' }

$people = @(
    @{ First='Dana';   Last='Whitfield';   Alias='dwhitfield';   Dept='Finance' }
    @{ First='Marcus'; Last='Reyes';       Alias='mreyes';       Dept='Finance' }
    @{ First='Priya';  Last='Nandakumar';  Alias='pnandakumar';  Dept='Finance' }
    @{ First='Ben';    Last='Achterberg';  Alias='bachterberg';  Dept='IT' }
    @{ First='Nadia';  Last='Farouk';      Alias='nfarouk';      Dept='IT' }
    @{ First='Elena';  Last='Mistry';      Alias='emistry';      Dept='Sales' }
)

foreach ($p in $people) {
    $upn = "$($p.Alias)@$domain"
    if (Get-MgUser -Filter "userPrincipalName eq '$upn'" -ErrorAction SilentlyContinue) { continue }

    $u = New-MgUser -DisplayName "$($p.First) $($p.Last)" `
                    -GivenName $p.First -Surname $p.Last `
                    -UserPrincipalName $upn -MailNickname $p.Alias `
                    -Department $p.Dept -UsageLocation US `
                    -AccountEnabled -PasswordProfile $pw

    Set-MgUserLicense -UserId $u.Id -AddLicenses @{SkuId = $e5} -RemoveLicenses @()
    Write-Host "Created and licensed $upn"
}
```

`-UsageLocation US` is set at creation because the Graph API requires it — unlike the
admin center, which quietly fills it from the tenant country, `Set-MgUserLicense` will
fail on a user with none. Usage location is a regulatory field determining which service
plans can be provisioned, not a billing address.

Mailboxes take a few minutes to provision after licensing. If `Get-Mailbox` comes back
empty right after this runs, that's normal — wait and retry rather than troubleshooting.

---

## 1. Reset a password

**M365 admin center** → Users → Active users → select user → Reset password.

Two choices on that screen. **Auto-generate** versus set one manually, and **require
this user to change their password when they first sign in**. Leave the change-on-signin
box checked unless the ticket says otherwise — an admin-set password that the admin also
knows is a shared credential until the user changes it.

```powershell
# Graph
$pw = @{ forceChangePasswordNextSignIn = $true; password = 'TempPass!2026x' }
Update-MgUser -UserId dana@contoso.com -PasswordProfile $pw
```

**Gotchas:**

- If it's a **hybrid** environment with password hash sync or pass-through auth, the
  cloud password reset either fails or gets overwritten on the next sync. The source of
  truth is on-prem AD. Ask whether the tenant is hybrid before you assume — in an MSP
  context this is a real and common trap, and asking the question is itself the right
  answer.
- Delivering the password matters. Don't email it to the address the user can't get
  into. Say out loud that you'd deliver it out-of-band and verify the requester's
  identity first, since password reset requests are the classic help desk social
  engineering vector.
- If the ticket is "user forgot password" and the tenant has SSPR enabled, the better
  answer may be pointing them at self-service rather than resetting it for them.

---

## 2. Assign the right licenses

**M365 admin center** → Users → Active users → select user → Licenses and apps.

This is the task most likely to be graded on precision rather than speed, because "the
right licenses" implies there's a wrong one.

**Know the Business SKUs.** Not to memorize — to reason from. In a lab you'll only be
choosing between whatever SKUs the tenant actually owns, which you can see under Billing
→ Licenses, and that's often just one or two. What you want in your head is the three
questions that discriminate between them: installed Office apps or web only, device
management needed or not, email only or the full suite.

| SKU | Desktop Office apps | Mailbox | Teams/SharePoint | Intune + Entra P1 + Defender |
|---|---|---|---|---|
| Business Basic | Web only | 50 GB | Yes | No |
| Business Standard | Yes, installed | 50 GB | Yes | No |
| Business Premium | Yes, installed | 50 GB | Yes | Yes |
| Exchange Online Plan 1 | No | 50 GB | No | No |
| Exchange Online Plan 2 | No | 100 GB + archive | No | No |

Business SKUs cap at 300 seats. Above that it's the enterprise line:

| SKU | Mailbox | Entra ID | Intune | Defender | Notes |
|---|---|---|---|---|---|
| Office 365 E1 | 50 GB, web apps only | Free | No | No | Frontline and light users |
| Office 365 E3 | 100 GB, desktop apps | Free | No | No | No identity or device management |
| Microsoft 365 E3 | 100 GB, desktop apps | P1 | Yes | Basic | Conditional Access included |
| Microsoft 365 E5 | 100 GB, desktop apps | P2 | Yes | Full stack | Adds Identity Protection, Defender for Identity, Purview |

The distinction that actually comes up on tickets: someone who only needs email gets
Exchange Online Plan 1, not Business Standard. Asking "what does this person actually
need" rather than assigning whatever everyone else has is the point of the exercise.

**Your tenant has E5.** The SKU part number is `SPE_E5`. Confirm before assuming:

```powershell
Get-MgSubscribedSku | Select-Object SkuPartNumber, SkuId, ConsumedUnits, PrepaidUnits
```

Common part numbers you'll see: `SPE_E5` for Microsoft 365 E5, `SPE_E3` for M365 E3,
`ENTERPRISEPACK` for Office 365 E3, `SPB` for Business Premium, `EXCHANGESTANDARD` for
Exchange Online Plan 1.

```powershell
# Usage location must be set first or assignment fails
Update-MgUser -UserId dwhitfield@SprawlDefense.onmicrosoft.com -UsageLocation US

$sku = Get-MgSubscribedSku | Where-Object SkuPartNumber -eq 'SPE_E5'
Set-MgUserLicense -UserId dwhitfield@SprawlDefense.onmicrosoft.com `
    -AddLicenses @{ SkuId = $sku.SkuId } -RemoveLicenses @()
```

**Gotchas:**

- **Usage location.** This is a regulatory field, not a billing address — Microsoft
  doesn't offer identical services in every country, so it determines which service plans
  can be provisioned to that user. The M365 admin center now **auto-fills it from the
  tenant's country** when the user has none, so portal assignment succeeds silently.
  Graph and PowerShell are stricter and still fail on a null usage location, which is why
  the seeding script sets it at creation. Group-based licensing falls back to the
  tenant's directory location rather than failing.
  The real risk isn't the error, it's the silent default: a client with staff in several
  countries gets everyone bucketed into the tenant's home country with no warning.
- **Group-based licensing** is how any competent MSP does this at scale — license
  attaches to a security group, membership drives assignment. Worth naming even if the
  ticket asks for a direct assignment: "I'd do this directly for one user, but at scale
  I'd use group-based licensing so it's driven by group membership rather than manual
  steps." If a user has no usage location under group-based licensing, assignment falls
  back to the tenant's directory location rather than hard-failing.
- **Service plans within a license can be toggled.** You can assign Business Premium but
  disable, say, Yammer or Sway for that user. The ticket may want this.
- **Removing a license starts a clock.** Mailbox data is retained 30 days after the
  license is removed, then it's gone. If the ticket is "downgrade this user," check
  whether anything needs preserving first.

---

## 3. Convert a user mailbox to a shared mailbox

**Exchange admin center** → Recipients → Mailboxes → select → Others → Convert to shared
mailbox.

```powershell
Set-Mailbox -Identity dana@contoso.com -Type Shared
```

**Gotchas:**

- **Order matters.** Convert first, *then* remove the license. A user mailbox needs a
  license to exist; a shared mailbox doesn't. Converting flips the mailbox type, so when
  the license comes off afterwards Exchange sees a shared mailbox and does nothing.
  Remove the license first and the mailbox disconnects instead, starting a 30-day
  retention clock, and there's no mailbox left in the list to convert.
- **It is recoverable, within 30 days.** Re-assign a license and the mailbox reconnects
  with data intact, then convert and delicense in the right order. The damage comes from
  nobody noticing — the ticket gets closed, everyone assumes the mail was kept, and the
  retrieval request arrives on day 45.
- **Converting does nothing for OneDrive.** Files are a separate retention setting in the
  SharePoint admin center. If the ticket says preserve their data, mail and files are two
  tasks, not one.
- A shared mailbox under **50 GB** needs no license. Over 50 GB, or if you want an
  online archive or litigation hold on it, it needs one. This is the cost argument that
  makes the conversion worth doing, and stating it shows you're thinking like someone
  who works for an MSP with clients who get invoiced.
- Conversion doesn't grant anyone access. You still need to add Full Access and, if they
  need to send from it, Send As. Those are separate permissions — see the scenario doc.
- The account should be blocked from sign-in. A shared mailbox with a sign-in-enabled
  account is a live credential nobody's watching.

---

## 4. Disable or enable an account

**M365 admin center** → Users → Active users → select user → Block sign-in (under the
account tab, or the three-dot menu).

```powershell
Update-MgUser -UserId dana@contoso.com -AccountEnabled:$false
Revoke-MgUserSignInSession -UserId dana@contoso.com
```

**Gotchas:**

- **Blocking sign-in does not end existing sessions.** Refresh tokens stay valid for up
  to an hour, sometimes longer. If the ticket is a termination or anything
  security-flavored, revoke sessions as well. Doing this unprompted is one of the
  clearest signals you've done this work before.
- Block sign-in, don't delete, unless the ticket explicitly says delete. Deletion
  soft-deletes for 30 days and starts clocks on the mailbox and OneDrive.
- If it's a termination, the full sequence is: block, revoke, reset password, convert
  mailbox to shared, reassign OneDrive ownership, remove from groups, then remove the
  license last.

---

## 5. Add a user to security groups

**M365 admin center** → Teams and groups → Active teams and groups → select group →
Members → Add members. Or from the user's page, Groups tab.

```powershell
$g = Get-MgGroup -Filter "displayName eq 'GRP-Finance'"
New-MgGroupMember -GroupId $g.Id -DirectoryObjectId (Get-MgUser -UserId dana@contoso.com).Id
```

**Know the group types**, because picking the wrong one is a real failure mode:

| Type | Has an email address | Used for | Notes |
|---|---|---|---|
| Security group | No | Permissions, licensing, CA scoping | The workhorse |
| Mail-enabled security group | Yes | Permissions *and* mail delivery | Can't be created in the M365 admin center UI, needs Exchange or PowerShell |
| Microsoft 365 group | Yes | Teams, SharePoint, shared inbox | Creates a SharePoint site and a group mailbox |
| Distribution list | Yes | Email distribution only | No permissions capability |

If a ticket says "add them to the security group so they get access to the share," a
security group is correct. If it says "and they should also receive the team's email,"
you need mail-enabled security or a M365 group instead, and noticing that distinction
is the whole test.

**Gotcha:** dynamic groups have membership driven by a rule, not manual addition. If
you try to add someone manually to a dynamic group the option is greyed out, and the fix
is editing the rule or the user's attributes — not the membership list.

---

## 6. Create a distribution list

**Exchange admin center** → Recipients → Groups → Add a group → Distribution list.

```powershell
New-DistributionGroup -Name 'Sales Team' -PrimarySmtpAddress sales@contoso.com `
    -Members @('dana@contoso.com','marcus@contoso.com') -Type Distribution
```

**The decision worth narrating.** Four things can receive mail at a shared address and
they are not interchangeable:

- **Distribution list** — mail fans out to members' individual inboxes. No shared
  history. Simple, and right when you just want to reach a group.
- **Shared mailbox** — one mailbox everyone opens. Shared history, shared sent items.
  Right for a support or info queue where continuity matters.
- **Microsoft 365 group** — shared inbox plus SharePoint plus Teams. Right when the
  group is a real collaborating team, heavier than needed otherwise.
- **Mail-enabled security group** — distribution plus permissions in one object.

Saying "a DL is what was asked for, though if they want shared history rather than
individual delivery a shared mailbox would fit better — want me to confirm with the
requester?" is a strong move. It shows you read the ticket for intent rather than
executing literally, which is exactly what an MSP wants.

**Gotchas:**

- New DLs default to accepting internal senders only. If external parties need to email
  it, you have to explicitly allow external senders. This is the most common
  "the distribution list doesn't work" follow-up ticket.
- Creation can take a few minutes to propagate before the address resolves.
- `-Type Distribution` versus `-Type Security` on `New-DistributionGroup` is what
  determines whether you get a plain DL or a mail-enabled security group.

---

## Working the ticket, not just the task

They're giving you a ticketing system on purpose. Assume it's scored.

**Read the whole ticket before touching anything.** Tickets contain more than the
request line — the requester's role, whether they're asking on someone else's behalf, a
due date, an attachment.

**Restate the ask out loud.** "This is asking me to convert Dana's mailbox to shared and
remove her license. Before I remove the license I want to do the conversion first, so
we don't start the retention clock." That single sentence demonstrates comprehension,
ordering, and awareness of a consequence.

**Verify, explicitly.** After every task, confirm the change actually took. Re-open the
user, check the license shows, check the group membership lists them. Say that you're
verifying rather than assuming.

**Write the resolution note.** What was requested, what you did, how you confirmed it,
and anything the next person needs to know. Two or three sentences. Most candidates skip
this entirely.

**Ask about scope when the ticket is ambiguous** rather than guessing. "The ticket says
add her to the security group but doesn't say which one — is there a naming convention
I should follow?" Asking is a scored behavior in every well-run lab.

---

## Drill order in SprawlDefense

Do each of these twice. The second run is where the speed comes from.

1. Run the seeding script. Six licensed users, two minutes.
2. Create a seventh user **in the portal by hand** and license them. The portal path
   matters as much as the script because the ticket will have you clicking. Note that
   the admin center fills in usage location for you from the tenant country — check the
   user's account page afterwards and see what it set. Then try the same thing through
   Graph without a usage location and see whether the API is as forgiving.
3. On that seventh user, assign E5 and then disable two service plans within it — turn
   off Viva Engage and Sway. Licenses and apps → expand the app list → uncheck.
4. Reset Dana's password with force-change-at-signin enabled.
5. Convert Marcus's mailbox to shared, in the correct order, then grant Dana Full Access
   and Send As. Confirm Dana can see it and could send as it.
6. Block Elena's sign-in, then revoke her sessions. Two separate actions.
7. Create a security group, add Ben and Nadia. Then create a distribution list and add
   the same two. Notice the different portals and what each object can do.
8. Try to create a mail-enabled security group in the M365 admin center, fail, then do
   it in Exchange. That failure teaches the portal boundary better than reading about it.
9. Delete the seventh user, then find it under Deleted users and restore it. Soft delete
   and the 30-day window are worth seeing once.

Roughly ninety minutes for the first pass, forty for the second.
