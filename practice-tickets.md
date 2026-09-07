# Practice tickets

Six tickets written the way real users write them. Work them against SprawlDefense,
write a resolution note for each, then check yourself against the key.

**Don't read Part 2 until you've worked all six.** The whole value is in reading the
ticket cold and deciding what's actually being asked.

**Time yourself.** Ten minutes per ticket including the note. If you're running long,
that's information.

---

# Part 1: the tickets

---

### TKT-4471 · Priority: Normal
**From:** Marguerite Okonjo, Office Manager
**Subject:** Sarah leaving

> Hi, Sarah Whitfield's last day was Friday. Can you go ahead and delete her account so
> we're not paying for it. Her manager Tom will need to be able to see her emails going
> forward since she was handling the vendor stuff. Thanks!

---

### TKT-4472 · Priority: Low
**From:** Devin Marsh, Sales Director
**Subject:** email address for the sales team

> Can we get a sales@ email set up as a distribution list. We want anything that comes
> into it to go to the whole team, and we need to be able to see what everyone has
> already replied to so we're not doubling up on the same customer. Currently we forward
> stuff to each other manually and it's a mess.

---

### TKT-4473 · Priority: High
**From:** Marguerite Okonjo, Office Manager
**Subject:** new starter

> We have a new person starting Monday, can you get them set up before then? Same as
> everyone else. Let me know when it's done.

---

### TKT-4474 · Priority: Urgent
**From:** Priya Nandakumar
**Subject:** LOCKED OUT - URGENT

> I got a new phone this weekend and now I can't get into anything, it keeps asking me
> for a code from an app that isn't on my new phone. I have a client presentation in 40
> minutes. Can you just turn off the two factor thing for my account so I can get in
> please. I'm on my personal laptop at home right now.

---

### TKT-4475 · Priority: Normal
**From:** Ben Achterberg, IT Coordinator
**Subject:** D drive full on FS01 again

> The D drive on the file server is at 98% again. There's a 500GB disk sitting in that
> server that nobody ever set up, can you just extend D onto it so we stop having this
> conversation every quarter.

---

### TKT-4476 · Priority: Normal
**From:** Devin Marsh, Sales Director
**Subject:** app server keeps changing address

> Our reporting server keeps changing its IP address and then nothing can connect to it
> until someone fixes it. Please set it to a static IP of 10.0.1.50 so this stops
> happening. It's the VM you'd be remoting into.

---

## Resolution note template

For each ticket, write four lines:

```
REQUESTED:
DONE:
VERIFIED:
NOTES / FOLLOW-UP:
```

Two sentences per line is plenty. The point is that someone picking this ticket up in
six months can tell what happened without asking you.

---

# Part 2: the key

Stop here if you haven't worked all six.

---

## TKT-4471 — termination

**What's actually being asked:** offboard Sarah while preserving her mail for Tom. The
requester said "delete" but that isn't what she needs, and doing it literally is the
wrong outcome.

**Traps:**
- "Delete her account" — deleting soft-deletes for 30 days and starts clocks. **Block
  sign-in instead.** Deletion is a separate decision the client should make deliberately.
- "So we're not paying for it" — the license question is real, and converting the mailbox
  to shared is what actually saves the money while keeping the mail. That's the answer
  to both halves of her request at once.
- **OneDrive isn't mentioned and she probably hasn't thought about it.** Sarah was
  handling vendor work; her files matter as much as her mail. Raise it.
- Revoking sessions isn't in the ticket and needs doing regardless.

**Sequence:** block sign-in → reset password → revoke sessions → convert mailbox to
shared → grant Tom Full Access (and Send As if he needs to reply as her) → assign Tom or
Marguerite as secondary OneDrive owner → remove group memberships → remove license last.

**Model note:**
```
REQUESTED: Offboard Sarah Whitfield, preserve mailbox access for Tom Okafor.
DONE: Blocked sign-in, reset password, revoked active sessions. Converted mailbox to
shared (no license required under 50GB) and granted Tom Full Access + Send As. Removed
M365 license. Account blocked, not deleted.
VERIFIED: Confirmed Tom can open the mailbox. Confirmed license shows unassigned and
mailbox still present in EAC.
NOTES: Account left in place rather than deleted — recommend confirming retention
requirements before deletion. Sarah's OneDrive has not been reassigned; flagged to
Marguerite as she handled vendor documentation. Awaiting decision on secondary owner.
```

**The differentiator:** raising OneDrive unprompted, and declining to delete.

---

## TKT-4472 — asking for the wrong thing

**What's actually being asked:** he says distribution list, but he describes wanting
shared history so the team doesn't double-reply. A DL fans out to individual inboxes —
it gives him exactly the problem he's trying to solve. **He wants a shared mailbox.**

**Traps:**
- Building what was asked for. It'll be back as a ticket in two weeks.
- Over-correcting silently. Don't just build a shared mailbox and say nothing; explain
  the difference and confirm.

**What to do:** reply on the ticket before building. "A distribution list delivers a copy
to each person's own inbox, so there's no shared record of who replied — which sounds
like the problem you're describing. A shared mailbox gives everyone one common inbox with
shared sent items, so you can see what's been answered. Want me to set that up instead?"

If you need to demonstrate the work rather than wait, build the shared mailbox, grant the
team Full Access and Send As, and note that you'd confirm with the requester.

**Model note:**
```
REQUESTED: sales@ distribution list for the sales team.
DONE: Clarified requirement with requester — the need for visible reply history points
to a shared mailbox rather than a DL, since a DL delivers individual copies with no
shared record. Created sales@ as a shared mailbox, granted Full Access and Send As to
the sales team security group.
VERIFIED: Test message to sales@ visible to all members. Confirmed sent items are shared.
NOTES: No license required while under 50GB. If external customers will email this
address directly, external senders may need to be permitted — not yet configured.
```

**The differentiator:** reading for intent, and asking rather than assuming.

---

## TKT-4473 — missing information

**What's actually being asked:** unanswerable as written. There is no name, no start
date beyond "Monday," no role, no department, no manager, no license specified, no
indication of what "same as everyone else" means.

**Traps:**
- Guessing. Creating an account with invented details is worse than asking.
- Asking one question at a time. Ask everything at once so it's one round trip.
- Treating "High priority, needs it Monday" as a reason to skip the questions. Urgency
  is a reason to ask *faster*, not to guess.

**What to do:** respond on the ticket immediately with a grouped list. Full name,
preferred display name and email format, job title and department, manager, start date,
which license, which groups or shared mailboxes they need, whether hardware is needed.
Set status to Pending / Awaiting Customer.

**Model note:**
```
REQUESTED: Provision a new user account before Monday.
DONE: Unable to proceed — ticket does not specify the user's name, role, department,
manager, or required access. Requested full details from Marguerite in ticket comments.
VERIFIED: N/A
NOTES: Status set to Awaiting Customer. Flagged the Monday deadline in the reply so the
requester understands the time constraint. Recommend a standard new-starter intake form
to avoid this round trip — happy to draft one.
```

**The differentiator:** this ticket is a test of whether you'll guess under time
pressure. Asking is the correct answer and it is being scored. Suggesting an intake form
is the bonus — it fixes the class of problem, not the instance.

---

## TKT-4474 — MFA, and a social engineering shape

**What's actually being asked:** she needs access. She's asked for the wrong remedy.

**Traps:**
- **Do not disable MFA.** This is the single clearest fail in the set. Removing the
  control to work around it, under time pressure, from a personal device, off-network.
- The request has the exact shape of a help desk social engineering attack: urgency, a
  named deadline, an unmanaged device, a request to weaken a security control. That
  doesn't mean it's an attack — it's probably a real panicked user. It means you verify.
- Don't lecture her. She's stressed and has a presentation in forty minutes.

**What to do:** verify identity out-of-band through whatever channel the client has
agreed to. Then issue a **Temporary Access Pass** — time-limited, single-use, satisfies
MFA once so she can sign in and register the new device herself. If TAP isn't enabled in
the authentication methods policy, enable it, or delete the stale Authenticator
registration and use "Require re-register multifactor authentication."

**Model note:**
```
REQUESTED: Disable MFA to restore access after a phone replacement.
DONE: Verified requester identity by callback to the number on file. Did not disable
MFA. Issued a one-time Temporary Access Pass valid for 60 minutes and walked Priya
through registering Microsoft Authenticator on her new device.
VERIFIED: Confirmed successful sign-in with the new method registered. Removed the stale
authentication method tied to the previous device.
NOTES: TAP was not previously enabled in the tenant's authentication methods policy —
enabled it, as this is the correct remedy for device-replacement lockouts and avoids
disabling MFA. Recommend users register a second method to reduce recurrence.
```

**The differentiator:** naming why you didn't disable MFA, and verifying identity without
being obstructive about it.

---

## TKT-4475 — disk extend

**What's actually being asked:** more space on D. "Just extend it onto the other disk"
is not something Disk Management can do.

**Traps:**
- **Extend Volume only works into unallocated space immediately to the right, on the
  same physical disk.** You cannot extend a basic volume onto a different disk. The
  option will be greyed out and no amount of clicking changes that.
- The 500 GB disk is probably uninitialized, so it won't even appear as usable until you
  initialize it.
- Don't convert to a dynamic disk to make spanning work. It's technically possible and
  it's a bad idea on a file server — it complicates recovery and is largely superseded.

**What to do:** initialize the new disk as GPT, create a volume, and either give it its
own letter or **mount it into a folder** on D so it extends the existing structure
logically. Then move a folder tree onto it. Ask which approach the client wants before
moving data.

**Model note:**
```
REQUESTED: Extend D: onto the unused 500GB disk in FS01.
DONE: D: cannot be extended onto a separate physical disk — Extend Volume only consumes
adjacent unallocated space on the same disk. Initialized the 500GB disk as GPT and
created an NTFS volume. Awaiting decision on presentation: separate drive letter, or
mounted into a folder under D: so existing paths stay intact.
VERIFIED: New volume online and formatted, 500GB available.
NOTES: Mounting into a folder avoids breaking mapped drives and shortcuts that point at
D:. Recommend identifying which folder tree to relocate before moving data. Longer term,
D: filling quarterly suggests a retention or quota conversation rather than repeated
capacity adds.
```

**The differentiator:** explaining *why* the ask isn't possible instead of just failing,
offering the mount-point alternative, and naming the recurring-capacity pattern.

---

## TKT-4476 — the dangerous one

**What's actually being asked:** he wants the server to be reliably reachable. He's
diagnosed it himself and prescribed a fix that is wrong and, on this VM, actively
harmful.

**Traps:**
- **This is the Azure VM you're RDP'd into.** Setting a static IP inside the guest OS on
  the primary adapter will drop your session and you will not get it back. Azure assigns
  the private IP via its own DHCP; the guest is expected to accept it.
- The right place to make an Azure VM's IP persistent is the **Azure NIC configuration**,
  where the private IP allocation is changed from Dynamic to Static. That's a control
  plane change, not a guest OS change.
- The actual underlying problem may not be the IP at all. If things connect by name, a
  DNS record or a DHCP reservation is the real fix.

**What to do:** do not make the change in the guest. Say why, out loud, clearly. Explain
where it should be done instead and what you'd need to do it.

**Say:** "Before I touch this — this is an Azure VM and I'm connected over RDP. Setting a
static address in the guest OS would drop my own session and I wouldn't get back in.
Azure allocates that IP through its own DHCP, so the correct fix is to set the private IP
allocation to Static on the VM's network interface in the Azure portal. And if things
connect to this server by name rather than address, a DNS record is probably the better
answer. How would you like me to proceed?"

**Model note:**
```
REQUESTED: Set a static IP of 10.0.1.50 on the reporting server.
DONE: Did not apply a static address in the guest OS. This VM is Azure-hosted and
receives its private IP from Azure's DHCP; configuring a static address inside Windows
would break connectivity and drop the RDP session. Correct remediation is to change the
private IP allocation to Static on the VM's network interface in Azure.
VERIFIED: Confirmed current allocation is Dynamic via ipconfig /all and the VM's network
settings.
NOTES: If clients connect by hostname, a DNS A record or DHCP reservation resolves this
without an IP change at all. Escalating for Azure portal access — I don't have
permissions on the subscription. Flagged to requester with the reasoning.
```

**The differentiator:** refusing to do the dangerous thing, explaining why in terms the
requester can follow, and offering the correct alternative. Recognizing that the
requester's self-diagnosis is wrong without being condescending about it.

---

## How to score yourself

| | |
|---|---|
| Did you read for intent, or execute literally? | 4471, 4472, 4476 all fail if executed literally |
| Did you ask when information was missing? | 4473 |
| Did you refuse the harmful ask? | 4474, 4476 |
| Did you write a resolution note at all? | All six |
| Did you note follow-ups nobody asked about? | OneDrive on 4471, external senders on 4472, retention on 4475 |

If you did all five, you'd pass this lab. The technical execution is the easy half.
