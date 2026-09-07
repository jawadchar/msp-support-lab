# M365 helpdesk scenarios

Nine scenarios an MSP support analyst hits constantly. For each one: how the user
phrases it, what to check and in what order, what usually turns out to be wrong, and
the thing that distinguishes a strong answer.

Scenarios marked **reproducible** can be built in the SprawlDefense trial tenant.
Scenarios marked **talk-through** need mail history or multiple licensed users that a
fresh trial does not have — know the path, do not burn time trying to stage them.

---

## Connecting

Both modules, every session. Exchange Online and Graph are separate connections with
separate auth.

```powershell
Install-Module ExchangeOnlineManagement -Scope CurrentUser
Connect-ExchangeOnline -UserPrincipalName Jawad@SprawlDefense.onmicrosoft.com

# Device code bypasses the WAM broker, which is what failed last time
Connect-MgGraph -Scopes 'User.Read.All','AuditLog.Read.All','Directory.Read.All' -UseDeviceCode
```

Request the scopes the task needs and no more. If asked why you didn't just grab
`Directory.ReadWrite.All`, the answer is that a consented scope persists on the service
principal until revoked, so an over-scoped admin tool is a standing privilege escalation
path. That reasoning transfers directly from least-privilege work you already do.

---

## Sign-in error codes worth knowing cold

Every sign-in failure in Entra carries a numeric code, and the code tells you which
layer failed. Reading them fluently is the fastest credibility signal available in an
identity troubleshooting scenario.

| Code | Means | Where to go next |
|---|---|---|
| 50126 | Wrong username or password | Password reset, or check for smart lockout |
| 50053 | Smart lockout, or account locked | Look for a spray pattern across accounts before assuming user error |
| 50055 | Password expired | Reset, check password policy |
| 50057 | Account disabled | Someone blocked sign-in, often an offboarding half-done |
| 50058 | Silent sign-in failed | Usually browser or session, not a real failure |
| 50074 / 50076 | MFA required, not satisfied | Authentication methods blade |
| 500121 | MFA attempted and failed, denied, or timed out | Repeated 500121 from odd locations is MFA fatigue, treat as an incident |
| 53000 | Device not compliant | Intune compliance, then the CA policy requiring it |
| 53001 | Device not joined | Entra join state, `dsregcmd /status` on the endpoint |
| 53003 | Blocked by Conditional Access | Sign-in log CA tab shows the exact policy |
| 65001 | User has not consented to the app | App consent, admin consent workflow |
| 700016 | App not found in directory | App registration or wrong tenant |

`53003` and `500121` are the two you will actually see most often, and they mean very
different things. A blocked-by-CA is working as designed. A run of failed MFA prompts
the user did not initiate is a push-fatigue attack.

---

## 1. "I'm not getting emails from our client" — talk-through

**Scope first.** One sender, or all external mail? One user, or everyone in the
department? Since when? Did they ever arrive? An MSP analyst who starts clicking before
establishing whether this is one mailbox or the whole tenant is showing you their
ceiling.

**Path:**

1. Message trace, Exchange admin center or `Get-MessageTrace -RecipientAddress
   dana@contoso.com -StartDate (Get-Date).AddDays(-2) -EndDate (Get-Date)`. This is the
   single most important tool in M365 support. It answers "did it even arrive" before
   you touch anything else.
2. Read the status. `Delivered` means it reached the mailbox and the problem is
   client-side or a rule. `FilteredAsSpam` or `Quarantined` means Defender caught it.
   `Failed` gives you an NDR to read. No result at all means it never reached Microsoft,
   which makes it the sender's problem, not yours.
3. If delivered but missing: `Get-InboxRule -Mailbox dana`. Rules that file to a
   subfolder or straight to Deleted Items are extremely common, and users forget they
   made them.
4. If quarantined: security.microsoft.com, Review, Quarantine. Release and, if it will
   recur, add a tenant allow-list entry rather than telling the user to check quarantine
   forever.
5. If nothing arrived: verify the recipient address is what the sender actually used.
   Typos and stale addresses in a sender's autocomplete cache are the single most common
   real cause.

**Ranked causes:** inbox rule the user forgot, quarantine, sender using a wrong or old
address, transport rule someone added months ago, mailbox at quota.

**What a weak candidate misses:** they investigate for twenty minutes without ever
running a message trace, and they never ask whether it's one sender or all mail. They
also skip inbox rules, which is the single highest-yield check on a "delivered but
missing" case.

---

## 2. "I got a new phone and now I can't log in" — reproducible

**Scope first.** Can they get in on a computer they're already signed into? Do they
still have the old phone? Do they have any other method registered?

**Path:**

1. Entra admin center, Users, the user, Sign-in logs. Confirm the failure is MFA and not
   a password problem. You are looking for 50074, 50076 or 500121 rather than 50126.
2. Authentication methods blade. See what's registered. If the Authenticator
   registration is bound to the old device, it is not going to work on the new one — the
   secret does not migrate unless they used Authenticator's cloud backup.
3. Fix it with a **Temporary Access Pass**. Generate a time-limited, optionally
   single-use passcode that satisfies MFA once, so the user can sign in and register the
   new device themselves. Entra, Users, Authentication methods, Add authentication
   method, Temporary Access Pass. TAP has to be enabled in the authentication methods
   policy first.
4. Fallback if TAP is not enabled: delete the stale method, or use "Require re-register
   multifactor authentication", then walk them through registration on a trusted network.

**What a weak candidate misses:** they reach for "just disable MFA for them temporarily."
That is the answer that ends the interview. Disabling MFA to fix an MFA problem opens a
window with no compensating control, and if the caller is a social engineer you have
handed them the account. TAP exists precisely so you never have to do that — it's
time-boxed, single-use if you want, and auditable.

Second thing they miss: verifying it's actually the user. In an MSP, a phone-based
identity reset request is the exact shape of a help desk social engineering attack. Say
out loud that you'd confirm identity through an out-of-band channel the client has
agreed to.

---

## 3. "I can't send from the shared mailbox" — reproducible

**Scope first.** Can they *read* it? Does it appear in Outlook automatically or did they
add it manually? What exact error?

**Path:**

Three permissions exist and they are independent. This is the whole scenario.

```powershell
# Read the mailbox and its contents
Add-MailboxPermission -Identity Support -User dana -AccessRights FullAccess -AutoMapping $true

# Send with the shared address as the From
Add-RecipientPermission -Identity Support -Trustee dana -AccessRights SendAs

# Send showing "Dana on behalf of Support"
Set-Mailbox -Identity Support -GrantSendOnBehalfTo @{Add='dana'}
```

Full Access alone lets them read and never send. That is the reported symptom about
ninety percent of the time.

Automapping is the second half. It only applies to Full Access granted through
`Add-MailboxPermission`, only works in Outlook desktop, and can take a few hours or an
Outlook restart to appear. If the user needs it now, they add the mailbox manually. If
you *don't* want it auto-mapping into everyone's profile, grant with
`-AutoMapping $false` — relevant for large shared mailboxes that would bloat the OST.

**Ranked causes:** Full Access granted but Send As forgotten, automapping not propagated
yet, user in Outlook Web where the mailbox has to be opened explicitly, group-based
permission where the group membership hasn't replicated.

**What a weak candidate misses:** the Send As versus Send on Behalf distinction. Send As
makes the message look like it came from the shared mailbox with nothing revealing who
sent it. Send on Behalf shows both names. Some clients specifically want the second one
for accountability, and some specifically want the first for a support queue. Knowing
there's a business decision embedded in the permission choice is the differentiator.

---

## 4. "The person who left still has access" or "we lost their files" — talk-through

**Scope first.** Was the account deleted or just blocked? When? Does anyone need the
mailbox contents? Does anyone need the OneDrive?

**Correct offboarding order:**

1. Block sign-in. Stops new authentications.
2. Revoke sessions: `Revoke-MgUserSignInSession -UserId dana@contoso.com`. This is the
   step people skip, and skipping it means the user keeps working for up to an hour on
   an existing refresh token. Blocking sign-in does not kill live sessions.
3. Reset the password. Order matters — reset then revoke, or revoke after reset, but
   never revoke first and reset later, because a token issued in between survives.
4. Remove the mobile device or wipe it in Intune if company data is on it.
5. Convert the mailbox to a shared mailbox, then remove the license. A shared mailbox
   under 50 GB needs no license, so this preserves the mail at zero cost.
6. Assign a secondary OneDrive owner, usually the manager, before deletion.
7. Remove group memberships and any delegated access they held elsewhere.

**The countdown nobody mentions:** removing a license from a still-existing account puts
the mailbox into a 30-day grace period, after which the data is unrecoverable. Deleting
the account soft-deletes it for 30 days. OneDrive after account deletion is retained for
a default 30 days, configurable up to 3,650 in the SharePoint admin center. Every one of
those is a clock, and the "we need his files" call always comes on day 45.

**What a weak candidate misses:** session revocation entirely, and the license-removal
data clock. Both are invisible until they cost the client something.

---

## 5. "Our emails are going to their spam folder" — talk-through

**Scope first.** Every recipient, or one domain? Started when? Did anything change —
new marketing platform, new invoicing system, a domain move?

**Path:**

1. Check the three DNS records. SPF is a TXT record on the domain:
   `v=spf1 include:spf.protection.outlook.com -all`. DKIM is two CNAMEs,
   `selector1._domainkey` and `selector2._domainkey`, and has to be enabled in the
   Defender portal, not just published. DMARC is a TXT record at `_dmarc`:
   `v=DMARC1; p=quarantine; rua=mailto:...`.
2. Read the headers of a message that landed in junk. `Authentication-Results` tells you
   which of spf, dkim and dmarc passed or failed. This is the fastest single check.
3. If SPF fails but DKIM passes, suspect forwarding. SPF breaks on forwarding because
   the forwarding server is not in the original domain's SPF record. DKIM survives
   forwarding because it signs the message itself. That asymmetry is the whole reason
   DKIM exists.
4. If a third-party sender is failing, they are probably not in the SPF record. Note the
   ten-DNS-lookup limit — an SPF record with too many `include:` statements fails as
   `permerror`, and every additional SaaS vendor pushes toward that ceiling.

**What a weak candidate misses:** they say "add them to the safe senders list," which is
a per-recipient bandage that does nothing for the other 400 people the client emails.
The fix is at the sending domain.

Your header-analysis background is directly relevant here. The trust boundary concept —
that received headers are only trustworthy from the first server you control inward —
is the same reasoning.

---

## 6. "I'm being blocked and I don't know why" — reproducible

**Scope first.** One app or all of them? One location or everywhere? Personal device or
corporate?

**Path:**

1. Entra, Sign-in logs, find the failure. Error 53003 means Conditional Access.
2. Open the sign-in entry and go to the **Conditional Access** tab. It lists every
   policy evaluated and the result for each: Success, Failure, Not applied, or Notice.
   The one showing Failure is your answer, by name. No guessing required.
3. Use **What If** (Entra, Conditional Access, What If) to model the user, app, device
   state and location and see what would apply. This is how you test a hypothesis
   without changing anything.
4. Decide whether the policy is correct and the user is out of scope legitimately, or
   whether the policy is over-broad. Do not just exclude the user — that's how
   exclusion lists grow until the policy protects nobody.

**Ranked causes:** device not compliant or not joined (53000, 53001), sign-in from an
untrusted location, legacy authentication client hitting a block-legacy-auth policy, an
app not in the policy's include list.

**What a weak candidate misses:** they conflate Conditional Access with role
permissions. CA decides whether you're allowed to authenticate at all under these
conditions. RBAC decides what you can do once you're in. They're separate layers
evaluated at different times, and "the user has the right role but is still blocked" is
a CA problem, not a permissions problem.

Second thing: report-only mode. Every CA policy should be deployed in report-only first
so you can see what it would have blocked. That's the same pattern as `-WhatIf` in
PowerShell and audit mode in an attack surface reduction rule — a cross-product theme
worth naming out loud.

---

## 7. "I can't open the file everyone else can see" — talk-through

**Scope first.** Teams, SharePoint or OneDrive? Did they ever have access? Is this one
file or the whole site?

**Path:**

1. Establish where the file actually lives. A file in a Teams standard channel lives in
   the team's SharePoint site, and access is governed by the underlying Microsoft 365
   group membership. A file in a private channel lives in a *separate* SharePoint site
   with its own membership. A file shared from someone's OneDrive is governed by that
   share link and nothing else. Getting this wrong sends you to the wrong permissions
   model entirely.
2. Check group membership first, since that's the cause most of the time.
3. If it's a link-based share, check the link type. "Anyone", "People in your
   organization", and "Specific people" behave completely differently, and a link that
   worked when forwarded internally will fail for a guest.
4. Check the SharePoint site's sharing setting and the tenant-level external sharing
   setting. The tenant setting is a ceiling — a site cannot be more permissive than the
   tenant allows, which is why "I set the site to allow guests and it still fails" is a
   common ticket.
5. Sensitivity labels can block access independently of permissions. If a label with
   encryption is applied, the label's own rights assignment governs, and site
   permissions won't override it.

**What a weak candidate misses:** the private channel distinction, and the tenant-level
ceiling. Both produce "I granted access and it still doesn't work," which is the most
frustrating class of ticket to be on the wrong side of.

---

## 8. "The new laptop won't connect to company resources" — reproducible

**Scope first.** Is the device enrolled? Entra-joined, hybrid-joined, or registered?
What does the error say?

**Path:**

1. On the device: `dsregcmd /status`. Read the Device State block.
   `AzureAdJoined: YES` means Entra-joined. `DomainJoined: YES` alongside it means
   hybrid. `WorkplaceJoined: YES` alone means registered only, which is a much weaker
   state and often the actual problem. Also read `TenantName` — a device joined to the
   wrong tenant is a real thing at an MSP with many clients.
2. Intune, Devices. Confirm it appears, and check compliance status. A device that isn't
   there was never enrolled.
3. If non-compliant, open the compliance policy and see which specific setting failed —
   BitLocker not enabled, OS build below minimum, Defender not running.
4. Tie it back to Conditional Access. Error 53000 at sign-in plus a non-compliant device
   in Intune is one problem, not two.

**What a weak candidate misses:** they never run `dsregcmd /status`, so they spend the
scenario guessing at join state. It's ten seconds and it settles the question. Learn to
read its output cold — on interview day it's likely your first command on any box they
hand you, because it tells you what kind of environment you're in before you touch
anything.

---

## 9. "Something's wrong with my account, people are getting weird emails from me" — reproducible

This is the one where your background is worth more than a service desk candidate's, so
be ready to take it further than they expect.

**Scope first.** Do not lead with reassurance. Treat it as compromise until disproved.

**Containment order:**

1. Block sign-in.
2. Reset the password.
3. `Revoke-MgUserSignInSession -UserId dana@contoso.com`. Password reset alone does not
   invalidate existing refresh tokens. If you revoke before resetting, a token issued in
   the gap survives. Reset, then revoke.

**Then hunt for persistence**, which is the part junior analysts skip entirely because
they think containment is the end of the job:

```powershell
Get-InboxRule -Mailbox dana | Select-Object Name, Enabled, MoveToFolder, ForwardTo, DeleteMessage
Get-Mailbox dana | Select-Object ForwardingSmtpAddress, ForwardingAddress, DeliverToMailboxAndForward
```

Four artifacts to check, in order of how often they're missed:

- **Inbox rules.** Attackers create rules that move replies and NDRs to RSS Feeds or
  Deleted Items so the user never sees the fraud thread. Rules named with a single
  character or a period are a classic tell.
- **Mail forwarding**, both mailbox-level and rule-level. They're configured in
  different places and checking one does not cover the other.
- **OAuth consent grants.** A malicious app with delegated mail permissions keeps its
  access after the password reset, because it isn't using the password. Entra,
  Enterprise applications, and review recent consents.
- **Registered MFA methods.** If the attacker added their own authenticator or phone
  number, they can pass MFA after your reset. Check the authentication methods blade and
  remove anything the user doesn't recognize.

**Then scope the blast radius:** `Search-UnifiedAuditLog` in Purview for what the account
did while compromised, and Entra sign-in logs for other accounts hit from the same IP or
matching a spray pattern.

**The AiTM point worth raising:** modern phishing kits proxy the real login page and
steal the session cookie after MFA completes. The victim sees a legitimate Microsoft
prompt and approves it honestly. This is why MFA alone isn't sufficient and why session
revocation is mandatory in the response — the attacker holds a valid token, not a
password. Saying this in an MSP interview signals you understand the current threat
model rather than the 2018 one.

---

## Three cross-cutting points to have ready

**Report-only, WhatIf, audit mode.** Conditional Access report-only, PowerShell
`-WhatIf`, Intune policy in report mode, Defender ASR rules in audit mode. Same idea in
four products: prove what a change will do before it does it. Naming the pattern rather
than the individual features reads as someone who thinks in principles.

**Blocking is not revoking.** Blocking sign-in stops new authentication. Existing
refresh tokens keep working until they expire. Revocation is a separate action and it's
the one people forget. This applies to offboarding, compromise response, and any
"we've disabled them but they're still in there" ticket.

**Two independent permission layers, always.** NTFS and share on a file server. Entra
role and resource RBAC in Azure. Conditional Access and application permissions in M365.
Mailbox Full Access and Send As in Exchange. The effective result is the intersection,
and troubleshooting means checking both rather than assuming one. This is the same
sentence you'll use on the file share exercise in the AD lab, which makes it a good
thread to pull across both halves of the interview.

---

## What to actually rehearse in SprawlDefense

Two hours, in this order:

1. Scenario 3, shared mailbox permissions. Create a shared mailbox, grant Full Access
   only, confirm the send failure, then add Send As. Ten minutes and it makes the
   distinction permanent.
2. Scenario 2, the TAP flow. Enable Temporary Access Pass in the authentication methods
   policy, issue one, use it. Most candidates have never seen this.
3. Scenario 6. Create a CA policy in report-only, run What If against it, then read the
   Conditional Access tab of a real sign-in.
4. Scenario 9, partially. Create an inbox rule that forwards externally, then find it
   with `Get-InboxRule`. Practice the containment order out loud even though you have no
   real compromise to respond to.
5. Scenario 8. Run `dsregcmd /status` on your own machine and read every line of the
   Device State block until it's familiar.

Scenarios 1, 4, 5 and 7 you can only talk through in a trial tenant. Know the paths,
don't burn build time on them.
