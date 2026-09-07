# Windows GUI drills

Three areas named in the interview brief: Device Manager, Disk Management, and network
configuration through Control Panel. These are execution tasks under observation. You
are being scored on knowing where things live and narrating what you're doing, not on
solving anything clever.

---

## Practicing without building a VM

Disk Management is the only one of the three that needs hardware you don't have. You can
fake it on your own machine with zero risk, because Disk Management can create and mount
virtual disks natively.

**Disk Management → Action → Create VHD.** The Location field wants a full path to a
**new** file that Windows will create — something like `C:\labdisks\lab1.vhdx`. Make the
folder first; the file must not already exist. Set the size to 5 GB, choose VHDX, and
pick dynamically expanding so the file starts near zero and only grows as you write to
it.

The new disk appears at the bottom of the disk list as Not Initialized with 5 GB
unallocated — exactly the state a freshly attached physical disk arrives in. It behaves
like real hardware for initialize, partition, format, extend and shrink.

**Create VHD** makes a new file. **Attach VHD**, directly below it in the same menu,
mounts a `.vhdx` that already exists — use that if you detach one and want it back later
without redoing the partitioning.

When you're finished, Action → Detach VHD on each, then delete the folder.

Make three of them. That covers every drill below without touching a real partition.

Device Manager and network config you can practice directly on your host, as long as you
put settings back.

---

## Part 1: Network configuration

They specifically mentioned Network and Sharing Center, so practice the Control Panel
path even though there are faster routes.

**Click path:** Control Panel → Network and Internet → Network and Sharing Center →
Change adapter settings → right-click the adapter → Properties → Internet Protocol
Version 4 (TCP/IPv4) → Properties.

**Faster route:** Win+R, `ncpa.cpl`. Drops you straight at Change adapter settings.
Use the Control Panel path in the lab if the ticket implies it, but know this one.

### The IPv4 properties dialog

Two radio button pairs. Address settings on top, DNS below, and they're independent —
you can take an address from DHCP while setting DNS manually, which is a common
configuration and a common source of confusion.

| Field | What it does |
|---|---|
| IP address | This host's address |
| Subnet mask | Which addresses are local versus needing the gateway |
| Default gateway | Where non-local traffic goes. Must be inside your own subnet |
| Preferred DNS | First server queried |
| Alternate DNS | Queried only if the preferred one doesn't answer |

**The Advanced button** is where the depth is, and most candidates never open it. Three
tabs: IP Settings holds multiple IP addresses and multiple gateways with metrics, DNS
holds a full ordered list beyond just two servers plus DNS suffix configuration, and
WINS is legacy. If a ticket says "add a third DNS server," Advanced → DNS → Add is the
answer, and it's the only place you can do it.

### Concepts to be able to say out loud

**Subnet mask.** It splits the address into network and host portions, and that split
determines whether the machine sends traffic directly or hands it to the gateway. A
wrong mask produces the distinctive symptom of reaching some hosts but not others — if
you're a /24 that should be a /16, everything in your own third octet works and nothing
else does. That specific symptom pattern is worth recognizing on sight.

**The gateway must be inside your subnet.** Windows will reject a gateway that isn't,
because the machine has no way to reach it. If someone typed the wrong mask, a
previously valid gateway can become unreachable without anyone touching it.

**DNS order is not load balancing and not failover on failure.** Windows queries the
preferred server. It only falls to the alternate if the preferred one *doesn't respond
at all* — a timeout. If the preferred server responds with NXDOMAIN, that's a valid
answer and the alternate is never consulted. This matters constantly: a DNS server
that's up but has the wrong or missing zone breaks resolution permanently, and adding a
good alternate does not fix it. This is a genuinely strong thing to know and most
candidates get it wrong.

### Verification

Never assume a change took. After every network change:

```
ipconfig /all
ping <gateway>
nslookup <internal name>
nslookup <internal name> <specific DNS server>
```

The two-form nslookup is the important habit. Querying by default tells you whether
resolution works. Querying a named server tells you whether *that server* works, which
separates a client misconfiguration from a server problem in one command.

If you change DNS, `ipconfig /flushdns` before testing, or you'll be reading a cached
answer and concluding your fix didn't work.

### Common ticket shapes

- "Can't reach internal resources but the internet works" — DNS pointing at a public
  resolver instead of the internal one.
- "Add a secondary DNS server" — Advanced → DNS if they want a third, the Alternate
  field if a second.
- "This machine needs a static IP" — check that the address is outside the DHCP scope
  range first, or you've created a future conflict.
- "Some sites work, others don't" — subnet mask.

### Azure warning, and this is the important one

**If the VM you're RDP'd into is in Azure, be extremely careful changing IP settings on
the primary adapter.** Azure assigns the VM's private IP through its own DHCP, and the
guest OS is expected to accept it. Setting a static IP inside the guest that doesn't
exactly match what Azure allocated will drop your RDP session, and you will not get it
back. You'd be locked out of your own interview.

DNS changes inside the guest are safe. Address, mask and gateway changes on the primary
NIC are not.

If a ticket in an Azure VM asks you to set a static IP, the correct answer is that the
address is reserved at the Azure NIC level, not in the guest — and saying that is a much
better answer than doing it. If they insist, say what you're about to do and why it's
risky before you do it. Narrating the risk protects you either way.

---

## Part 2: Disk Management

**Open:** Win+X → Disk Management, or Win+R → `diskmgmt.msc`.

### Initializing a new disk

A newly attached disk shows as **Not Initialized**. Right-click the disk number on the
left → Initialize Disk. It asks MBR or GPT.

| | MBR | GPT |
|---|---|---|
| Max disk size | 2 TB | Effectively unlimited |
| Partitions | 4 primary, or 3 + extended | 128 in Windows |
| Boot requires | Legacy BIOS | UEFI |
| Redundancy | Single partition table | Backup table at end of disk |

**Choose GPT** unless something specifically needs MBR. For a data disk on a modern
system it's the right default, and being able to say why — over 2 TB support, more
partitions, a backup partition table — is a clean thirty-second answer.

### Creating a partition

Right-click the unallocated space → New Simple Volume. The wizard asks size, drive
letter, and format.

Format options worth understanding: **NTFS** for Windows volumes, always, since it's the
only one with permissions, journaling and large file support. **exFAT** for removable
media shared with macOS. **FAT32** only when something legacy demands it, and note its
4 GB per-file limit. Leave allocation unit size at default unless the ticket specifies.

**Quick format versus full.** Quick writes a new file table and takes seconds. Full
additionally zeroes and scans every sector, and takes a long time on a large disk. Quick
is right for a new disk. Full is for a disk you suspect is failing, or one being
repurposed where you want the old data actually gone.

### Extending a volume

This is the drill most likely to appear, and it has one trap.

**Extend Volume is greyed out unless there is unallocated space immediately to the right
of the partition on the same physical disk.** Not elsewhere on the disk, not on another
disk, not to the left. Adjacent and following.

That's why "the D drive is full, extend it" sometimes just doesn't work — there's a
recovery partition sitting between D and the free space. The honest answer in that case
is that you can't extend past it without third-party tooling or moving the partition,
and knowing that is better than clicking around hoping the option lights up.

### Shrinking a volume

Right-click → Shrink Volume. Windows tells you the maximum shrinkable amount, and it's
often far less than the free space suggests. The reason is unmovable files — the page
file, hibernation file, and volume shadow copies sit at fixed locations, and Windows
won't shrink past them. Disabling the page file and shadow copies temporarily is the
workaround, but for a lab just know the explanation.

### Other things to be able to find

- **Change Drive Letter and Paths** — rename a drive letter, or mount a volume into an
  empty NTFS folder instead of giving it a letter.
- **Disk shows Offline** — right-click the disk → Online. On Windows Server the default
  SAN policy is Offline Shared, so newly attached disks come up offline deliberately.
  `diskpart` → `san policy=onlineall` changes it.
- **Foreign disk** — a dynamic disk from another machine. Right-click → Import Foreign
  Disks.
- **Basic versus dynamic** — basic is the normal case. Dynamic supports software
  spanning and mirroring and is largely superseded by Storage Spaces. Converting basic
  to dynamic is easy; converting back requires deleting all volumes.

---

## Part 3: Device Manager

**Open:** Win+X → Device Manager, or Win+R → `devmgmt.msc`.

### First move, every time

**View → Show hidden devices.** This reveals disconnected and ghost devices that are
still registered. A stale NIC entry holding a static IP configuration is a real and
maddening problem, and it's invisible until you turn this on. Doing it reflexively looks
like experience.

### Reading the icons

- **Yellow triangle** — the device has a problem. Open Properties and read the Device
  status box, which gives you a plain-language message and a code.
- **Down arrow on the icon** — disabled, by a person or by policy. Not broken.
- **Under "Other devices" with a generic name** — Windows sees hardware and has no
  driver for it.

### Error codes worth recognizing

| Code | Means |
|---|---|
| 10 | Device cannot start. Driver or hardware fault |
| 28 | Drivers not installed. The classic "Other devices" entry |
| 31 | Device not working properly, driver load failed |
| 43 | Windows stopped the device because it reported a problem |
| 45 | Not currently connected. A ghost device, only visible with hidden devices shown |

### The four actions

**Update driver** — search automatically, or browse to a file. In a lab with no internet
the automatic search fails, which is expected, not an error you caused.

**Roll Back Driver** — Properties → Driver tab. Greyed out unless a previous driver is
stored, which only happens if the current one was an update over an existing driver.
This is the correct answer to "it worked until the update," and reaching for it directly
rather than reinstalling is the differentiator.

**Disable / Enable** — for isolating whether a device is causing a problem.

**Uninstall device** — with a checkbox to delete the driver software. Leaving it
unchecked means Windows reinstalls the same driver on the next scan, which is fine for a
corrupted install and useless if the driver itself is the problem. Follow with **Action →
Scan for hardware changes** to trigger redetection.

### Identifying an unknown device

Properties → Details tab → Property dropdown → **Hardware Ids**. You get something like
`PCI\VEN_8086&DEV_1502`. The VEN is the vendor and the DEV is the device, and those two
numbers identify the hardware exactly. Searching them finds the driver.

Knowing this exists is worth real credit. It's the difference between "I'd look for a
driver" and "I'd pull the hardware ID and identify the device from the vendor and device
codes."

---

## The drills

Do these with the VHDs you created. Narrate out loud through all of them.

**Disk Management**

1. Create three VHDs. Initialize one as MBR and one as GPT. Say the difference aloud.
2. Create a simple volume on one, NTFS, assign a letter, quick format.
3. Create a small volume that leaves unallocated space after it, then extend it into
   that space.
4. Create a volume that fills the disk, then try to extend it. Watch the option grey
   out and explain why.
5. Shrink a volume. Note how much less than the free space it offers.
6. Change a drive letter, then mount a volume into an empty folder instead.
7. Take a disk offline and bring it back online.

**Network**

8. Walk the full Control Panel path to IPv4 properties without shortcuts. Twice.
9. Note your current settings, switch to a static IP matching them, verify connectivity
   still works, switch back to DHCP.
10. Open Advanced → DNS and add a third DNS server. Verify with `ipconfig /all`.
11. Set your DNS to something nonexistent like 10.99.99.99, watch resolution fail,
    verify with `nslookup`, then fix it. Flush the cache before retesting.
12. Say out loud why an alternate DNS server doesn't rescue you from a preferred server
    returning NXDOMAIN.

**Device Manager**

13. Turn on Show hidden devices. Look at what appears.
14. Disable a non-critical device, observe the icon change, re-enable it.
15. Open a network adapter's Properties and find the Driver tab. Note whether Roll Back
    is available and why.
16. Pull the Hardware Ids for any device and read the VEN and DEV values.
17. Uninstall a non-critical device without deleting the driver, then Scan for hardware
    changes and watch it return.

Roughly ninety minutes for all seventeen. Second pass is much faster and that's where
the fluency comes from.

---

## Narration lines

Have these ready. They cost nothing and they're what's being scored.

Before acting: "I'm going to check the current IP configuration before I change
anything, so I have a known state to go back to."

On scope: "Is this affecting just this machine, or other users too?"

On a greyed-out option: "Extend is unavailable here, which tells me there's no adjacent
unallocated space on this disk. Let me confirm that in the layout view."

On verification: "That's applied. Let me confirm the original symptom is actually gone
rather than assume it."

On risk, especially in Azure: "Before I change the address on this adapter — this is an
Azure VM and I'm connected over RDP, so a wrong static IP would drop my own session.
Is a static address definitely what's wanted here, or is DNS the actual ask?"

When stuck: "I haven't found it yet. Here's what I've ruled out, and here's what I'd
check next."

That last one is a passing answer even if you never solve the problem. Silence isn't.
