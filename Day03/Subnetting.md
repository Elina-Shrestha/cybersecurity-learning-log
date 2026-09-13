# Subnetting

An IP address without a mask is incomplete information. It tells you *which
machine*, but not which network it belongs to, how big that network is, or
where it stops. Before running something like `nmap 192.168.56.0/24`, I
should be able to say out loud how many addresses that touches, where the
range starts, and where it ends.

---

## Part 1: An address is 32 bits
- An IPv4 address is four numbers (octets), each 0–255, separated by dots.
- Each octet is 8 bits. 4 × 8 = 32 bits total — this is why IPv4 is called
  "32-bit addressing."
- 8 bits can hold 256 different values (0 through 255), which is why
  `192.168.1.300` is invalid — 300 doesn't fit in a single octet.
- Total possible IPv4 addresses: 2³² = 4,294,967,296 — which is why the
  world ran out of public IPv4 addresses in the 2010s.

**Reading binary into decimal** — each bit position is worth double the one
to its right:

| Bit position | 128 | 64 | 32 | 16 | 8 | 4 | 2 | 1 |
|---|---|---|---|---|---|---|---|---|

Example: `11000000` → 128 + 64 = **192**

**Why subnet masks always look like 255, 254, 252, 248, 240, 224, 192, 128,
or 0**: a mask's 1-bits always start from the left and run unbroken — never
scattered (like `10101010`). Once the 1s stop, the rest are 0s. That's why
only nine possible values exist per octet: 0, 128, 192, 224, 240, 248, 252,
254, 255.

---

## Part 2: The mask splits the address
Think of it like a postal address: `House 12, New Baneshwor, Kathmandu`.
"New Baneshwor" gets the letter to the right neighborhood; "House 12" only
means something *within* that neighborhood — there's a House 12 in every
other neighborhood too.

Same idea with `192.168.1.77 /24`: `192.168.1` is the neighborhood
(network), `77` is the house (host). The mask is the **only** thing that
tells you where the network part ends and the host part begins — the IP
address alone can't tell you that.

**CIDR vs dotted-decimal — same thing, two notations:**
- `/24` = CIDR notation → count of network bits (what Nmap expects)
- `255.255.255.0` = dotted-decimal → the same 32 bits written as 4 octets
  (what Windows/routers show)

**Important correction to a common assumption**: `/24` does NOT mean 24
addresses. It means 24 *network bits*, leaving 8 *host bits* → 2⁸ = 256
addresses.

**Common mask sizes:**

| CIDR | Mask | Host bits | Total addresses | Usable | Typical use |
|---|---|---|---|---|---|
| /8 | 255.0.0.0 | 24 | 16,777,216 | 16,777,214 | Huge private range (10.x.x.x) |
| /16 | 255.255.0.0 | 16 | 65,536 | 65,534 | Campus / large company |
| /24 | 255.255.255.0 | 8 | 256 | 254 | Default almost everywhere |
| /25 | 255.255.255.128 | 7 | 128 | 126 | Half of a /24 |
| /26 | 255.255.255.192 | 6 | 64 | 62 | Quarter of a /24 (a department) |
| /27 | 255.255.255.224 | 5 | 32 | 30 | Small office / VLAN |
| /28 | 255.255.255.240 | 4 | 16 | 14 | A rack / DMZ |
| /29 | 255.255.255.248 | 3 | 8 | 6 | Tiny ISP block |
| /30 | 255.255.255.252 | 2 | 4 | 2 | Point-to-point link between routers |

**Private (non-routable) ranges I'll actually see in labs:**
- `10.0.0.0/8` — large enterprises, cloud VPCs
- `172.16.0.0/12` — Docker's default bridge, mid-size networks
- `192.168.0.0/16` — home routers, VirtualBox/VMware lab networks
- Also: `127.0.0.0/8` = loopback (localhost), `169.254.0.0/16` = self-assigned
  when DHCP fails

**Important**: private doesn't mean permitted to scan. Scanning a network
without authorization is unauthorized access, even if it's a private range.

---

## Part 3: Two addresses you never get to use
Every subnet reserves its **first** address (network address) and **last**
address (broadcast address) — neither can be assigned to a device.

- **Network address** (e.g. `192.168.1.0`) — names the subnet itself,
  nothing answers on it
- **Usable range** (e.g. `192.168.1.1`–`.254`) — what you can actually
  assign to devices
- **Broadcast address** (e.g. `192.168.1.255`) — sends to *every* device on
  the subnet at once

**The two formulas:**
- Total addresses = 2ʰ, where h = host bits (32 − CIDR number)
- Usable addresses = 2ʰ − 2

**Worked for a /26**: 32 − 26 = 6 host bits → 2⁶ = 64 total → 64 − 2 = 62
usable.

**Trap to remember**: "how many hosts fit in a /24?" → 254. "How many
addresses does `nmap` scan in a /24?" → 256 (Nmap scans the reserved ones
too, since it doesn't assume your mask is correct).

---

## Part 4: Finding the network from any address (the actual skill)

**The one rule: block size**

> Block size = 256 − the "interesting octet" of the mask
> (the octet that's neither 255 nor 0 — where the split happens)

Example: for a /26, the mask is `255.255.255.192`, so the interesting octet
is 192, and block size = 256 − 192 = **64**.

The block size tells me:
- How many addresses are in each subnet
- How far apart subnet boundaries sit
- Boundaries always start counting from 0

**The four-step recipe:**
1. `256 − mask octet` = block size
2. Count up in block sizes from 0 → these are the subnet boundaries
3. Find which boundary my address falls just above
4. Broadcast = next boundary − 1

Then always: **first usable = network + 1**, **last usable = broadcast − 1**.

### Worked Example 1 — `192.168.1.77 /26`
1. Mask `255.255.255.192` → interesting octet 192 → block size = 256−192 = 64
2. Boundaries: 0, 64, 128, 192
3. 77 is between 64 and 128 → falls in the block starting at 64
4. Next boundary is 128 → broadcast = 128 − 1 = 127

**Answer**: Network `192.168.1.64/26`, usable `.65`–`.126`, broadcast `.127`
(62 usable addresses)

### Worked Example 2 — `10.20.30.200 /27`
1. Mask `255.255.255.224` → block size = 256 − 224 = 32
2. Boundaries: 0, 32, 64, 96, 128, 160, 192, 224
3. 200 is between 192 and 224 → block starts at 192
4. Next boundary is 224 → broadcast = 223

**Answer**: Network `10.20.30.192/27`, first usable `.193`, last usable
`.222`, broadcast `.223` (30 usable addresses)

### Worked Example 3 — `172.16.5.130 /25`
1. Mask `255.255.255.128` → block size = 256 − 128 = 128
2. Boundaries: 0, 128
3. 130 is past 128 → block starts at 128
4. Next boundary would be 256 (doesn't exist in an octet) → broadcast = 255

**Answer**: Network `172.16.5.128/25`, usable `.129`–`.254`, broadcast
`.255` (126 usable addresses)

**Key visual takeaway**: `192.168.1.77` and `192.168.1.130` look like
neighbors (same first 3 octets) but under a /26 mask they're on
**different** networks and can't talk without a router — a genuinely
counter-intuitive but important point.

---

## Part 5: Splitting one network into several
Same rule, used forwards — instead of finding which subnet an address is
in, I choose where the boundaries go.

**Why split a network at all:**
- **Containment** — one compromised device on a flat /24 can reach 253
  others directly; splitting forces an attacker through a router with rules
- **Reduced broadcast noise** — broadcast traffic reaches everyone in a
  subnet; smaller subnets mean less wasted processing
- **Policy boundaries** — finance, guest Wi-Fi, and CCTV shouldn't share a
  subnet, since separate subnets allow separate firewall rules
- **Clarity** — "the camera VLAN is 192.168.10.192/26" is precise and
  actionable; "the cameras are somewhere in 192.168.10.x" is not

**Trade-off**: splitting a /24 into four /26s gives 4 × 62 = 248 usable
addresses total, vs. 254 in one /24 — I lose 6 addresses (2 per new
subnet) as the cost of the extra network/broadcast addresses each split
creates.

**Design approach**: figure out how many machines my biggest group needs,
pick the smallest subnet size that fits, then check I have enough subnets —
never the reverse.

---

## Part 6: Applying this to Nmap

**Five ways to specify a target:**
```
nmap -sL -n 192.168.1.0/24          # whole subnet, 256 addresses
nmap -sL -n 192.168.1.77/26         # Nmap rounds DOWN to .64, 64 addresses
nmap -sL -n 192.168.1.1-50          # octet range, 50 addresses
nmap -sL -n 192.168.1.1,10,20       # exact list, 3 addresses
nmap -sL -n -iL targets.txt         # from a file — how real scopes are handled
```
`--exclude 192.168.1.5` removes named hosts from any of the above.

**Why `-sL` matters**: List Scan — prints exactly which addresses Nmap
would target and sends **zero packets**. `-n` turns off DNS lookups. This
is the safest command in Nmap and should run before every real scan.

**Scan size scales fast** — a default Nmap scan probes 1,000 ports per
host:

| Target | Addresses | Port probes |
|---|---|---|
| /30 | 4 | 4,000 |
| /28 | 16 | 16,000 |
| /24 | 256 | 256,000 |
| /16 | 65,536 | 65,536,000 |

The difference between `/24` and `/16` is one character — and the
difference between a five-minute scan and one still running when the
client calls asking what's happening to their network.

**Why this is a legal issue, not just a technical one**: scope is defined
by the mask. Typing `/24` instead of an authorized `/26` means scanning 4×
more addresses than permitted — and every packet carries my source
address. In Nepal, the Electronic Transactions Act 2063, Section 45 covers
unauthorized access to a computer system — penalty up to 3 years
imprisonment or a fine up to NPR 200,000, or both (it's "or," not "and,"
and it doesn't require proof of damage).

**What protects me**: written authorization stating the exact CIDR blocks
I may touch, a `targets.txt` file built from that authorization (used with
`-iL`), and always running `-sL -n` first to see the target list before
sending a single packet.

---

## Four mistakes to watch for in myself
1. **Starting the boundary count at the block size instead of 0** — every
   answer comes out one block too high
2. **Forgetting to subtract 1 for broadcast** — writing the next boundary
   itself instead of boundary − 1
3. **Confusing 256 with 254** — 256 addresses exist, 254 are assignable;
   the question determines which number is right
4. **Assuming every network is a /24** — always check the real mask with
   `ip addr` instead of guessing

---

## Cheat sheet (the one I'd photograph)

**The four steps:**
1. `256 − mask octet` = block size
2. Count up in blocks from 0
3. Find the boundary just below my address
4. Broadcast = next boundary − 1
5. First usable = network + 1, Last usable = broadcast − 1

**Habit to build**: calculate by hand → confirm with `nmap -sL -n` → only
then scan. Every time, even when I'm sure.

---

## Practice problems I worked through

**Q1: `192.168.1.200 /28`**
Block size = 256−240 = 16. Boundaries: ...176, 192, 208. 200 falls in
block starting at 192. → Network `.192`, usable `.193`–`.206`, broadcast
`.207`, 14 usable.

**Q2: `10.10.10.35 /29`**
Block size = 256−248 = 8. Boundaries: 0,8,16,24,32,40. 35 falls in block
starting at 32. → Network `.32`, usable `.33`–`.38`, broadcast `.39`, 6
usable.

**Q3: `172.16.5.130 /25`**
Block size = 128. 130 is past 128 → upper half. → Network `172.16.5.128/25`
(the upper half of the /24).

**Q4: How many addresses does `nmap 10.0.0.0/22` target?**
32 − 22 = 10 host bits → 2¹⁰ = **1,024 addresses**.

**Q5: Authorized to test only `192.168.20.128/26` — the exact command:**
```
nmap -sL -n 192.168.20.128/26
```

---

## What I'd still want to double check
- Placing an address between two boundaries (step 3) is where I'm most
  likely to slip — worth drilling a few extra examples on paper
- The `/25` "next boundary is 256" edge case (where the boundary rolls
  over past the octet) — needs a second look until it feels natural