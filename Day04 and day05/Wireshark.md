# Wireshark Masterclass — Study Notes

*Notes from Day 9 of my cybersecurity course*

## Why this matters
Nmap and my port scanner tell me *whether* something is open. Wireshark
tells me *what's actually happening* on the wire — every byte flowing
between two machines. If subnetting is knowing which apartment building
someone lives in, Wireshark is reading their mail as it moves through the
postal system. This is the tool that turns "the connection dropped" from
a guess into a diagnosis.

---

## Part 1: Setting up a real capture environment

Before capturing anything, the OS needs to actually let a normal user
read raw network traffic (by default, only root/admin can):

**Linux (Debian/Ubuntu):**
```bash
sudo apt update
sudo apt install wireshark -y
sudo usermod -aG wireshark $USER
# then reboot — group permissions don't apply until the session restarts
```

**macOS:**
```bash
brew install wireshark
# also need ChmodBPF package installed to allow interface access
```

**Windows:** the installer needs the **Npcap** driver option checked —
without it, Wireshark can't actually capture packets at all, only read
saved files.

**Windows — preparing for TLS decryption later:**
```powershell
setx SSLKEYLOGFILE "%USERPROFILE%\Desktop\sslkeys.log"
```
This sets an environment variable that browsers check — if it's set,
Chrome/Firefox will *voluntarily* write out the encryption keys they
generate. This is the setup step for Part 6 below.

---

## Part 2: What's actually inside a packet
A packet isn't one blob — it's layered, like nested envelopes. Each layer
only cares about its own job:

| Layer | OSI equivalent | Contains | Why it matters |
|---|---|---|---|
| Ethernet | Layer 2 | Source & destination MAC addresses | Hardware-level identification; **never leaves the local network (LAN)** |
| IP | Layer 3 | Source & target IPs, TTL, protocol indicator | Moves packets between networks/the Internet; reveals spoofing or misconfigurations |
| TCP/UDP | Layer 4 | Source/destination ports, sequence/ACK numbers (TCP only) | Defines application routing and connection reliability |
| Payload | Layer 7 | HTTP/DNS requests, TLS encrypted data | The actual human-readable (or encrypted) content |

**Key realization**: a MAC address is meaningless once a packet leaves my
LAN — the router strips it and puts its own on. IP addresses are what
survive the whole journey across the internet.

---

## Part 3: Two totally different kinds of filter
This tripped me up initially — Wireshark actually has **two separate**
filtering systems, and mixing up their syntax causes nothing but errors.

| | Capture Filters ("the gatekeeper") | Display Filters ("the scalpel") |
|---|---|---|
| Engine | BPF (Berkeley Packet Filter) / libpcap | Wireshark's own internal engine |
| When it runs | **Before** capture — decides what gets written to disk | **After** capture — decides what's shown on screen |
| Used for | Saving disk space, isolating high-volume targets | Granular analysis, deep packet slicing, behavioral hunting |
| Syntax feel | English-like (`host`, `net`, `port`, `and`) | Object-oriented field comparisons (`ip.addr ==`, `tcp.port ==`) |

**The practical rule I'm taking from this**: if I already know exactly
what I don't want cluttering a huge capture (e.g., only care about one
host), use a **capture filter** to keep the file small from the start. If
I already have a capture and want to slice into it different ways without
re-capturing, that's a **display filter** — and I can change display
filters infinitely without losing any data, since they don't touch what's
saved to disk.

### Capture filter examples (BPF syntax)
```
host 192.168.1.100                          # single host, any direction
src host 10.0.0.5                           # only traffic FROM this host
dst host 8.8.8.8                            # only traffic TO this host
net 192.168.1.0/24                          # whole subnet
ether host 00:11:22:33:44:55                # by MAC address
tcp port 80 or udp port 53                  # specific ports, mixed protocol
tcp and (port 22 or port 80 or port 443)    # port list
portrange 1024-2048                         # port range
not port 5353                               # exclude noise (e.g. mDNS)
vlan 100 and host 192.168.1.10              # VLAN-aware capture
broadcast or multicast                      # only broadcast/multicast traffic
tcp[13] & 0x02 != 0                         # byte-offset match: TCP SYN flag only
```

### Display filter examples (Wireshark syntax)
```
ip.addr == 192.168.1.100                                   # any direction
(ip.addr == 10.0.0.5 and tcp) or (ip.addr == 10.0.0.6 and not tcp)
tcp.port in {22 80 443 3389}                                # port set
!(mdns or dhcp or arp)                                      # exclude LAN noise
http.request.method == "POST"                               # HTTP POSTs only
http.request.uri contains "/login"                          # specific URI
frame.len > 1500                                             # oversized frames
tcp.flags.syn == 1 and tcp.flags.ack == 0                    # connection attempts
tcp.flags.reset == 1                                         # dropped/reset connections
```

---

## Part 4: TCP vs UDP — what "healthy" looks like

| | TCP | UDP |
|---|---|---|
| Design | Reliable, connection-oriented (3-way handshake) | Fast, lightweight, connectionless |
| Key headers | Sequence numbers, ACKs, flags (SYN, FIN, RST) | Source/destination port only — no seq/ACK |
| What I'm checking for | Data arrives in order, without loss | Speed over reliability (streaming, DNS, gaming) |
| Diagnostic filters | `tcp.analysis.retransmission`, `tcp.analysis.duplicate_ack` | `udp.port == 53`, or just `udp` |

**Why TCP needs so much more diagnostic machinery than UDP**: TCP is
*promising* delivery, so Wireshark can actually detect when that promise
is broken (a retransmission means "I had to resend this, something went
wrong"). UDP makes no such promise, so there's nothing equivalent to
detect — a dropped UDP packet just silently vanishes, which is exactly
why it's used for things where a missed frame doesn't matter (a
video call frame) rather than things that must arrive intact.

---

## Part 5: Diagnosing TCP problems from symptoms

| Symptom | Filter to check | What it means |
|---|---|---|
| General network slowness | `tcp.analysis.retransmission or tcp.analysis.duplicate_ack` | Packet loss — data is dropping in transit, sender is being forced to resend. View graphically via **Statistics > TCP Stream Graphs > Time Sequence (Stevens)** |
| Connection suddenly drops | `tcp.flags.reset == 1` | A TCP Reset — the server or a firewall is forcefully killing the connection |
| Application freezes/hangs | `tcp.analysis.zero_window or tcp.window_size == 0` | Zero Window — the receiving server is overwhelmed/out of memory and literally cannot accept more data right now |

**The pattern I want to internalize**: symptom → filter → graphical
confirmation (if needed) → diagnosis. Guessing "the network is slow" is
not a diagnosis; finding *which* of these three filters lights up is.

---

## Part 6: Decrypting TLS to see inside HTTPS

Modern traffic is almost entirely encrypted, so seeing "Application Data"
in Wireshark by itself tells me nothing. To actually read it, I need the
session keys — not to break the encryption, but because the browser
already generated the key and I'm just asking it to also write down a
copy.

**The workflow:**
1. **The client (browser)** generates symmetric session keys during the
   TLS handshake
2. **The environment variable** (`SSLKEYLOGFILE`, set up in Part 1) tells
   the OS to intercept and log those keys
3. **The key log file** (`sslkeys.log`) receives the keys in real time as
   they're generated
4. **The Wireshark engine** ingests that log file, matches session IDs to
   the right TCP streams, and unlocks the payload to reveal plain text

**Configuring Wireshark itself:**
`Edit > Preferences > Protocols > TLS` → set the "(Pre)-Master-Secret log
filename" field to point at `sslkeys.log`.

**The result**: previously unreadable "Application Data" packets
immediately transform into readable HTTP requests, JSON payloads, and
clear-text headers — this works even on TLS 1.3, as long as I control the
client endpoint doing the logging (I can't decrypt someone else's traffic
this way, only my own machine's).

**Important limitation I want to remember**: this only works because I
control the browser generating the keys. This is not a way to break into
someone else's encrypted traffic — it only works on sessions where I
already have legitimate access to the client side.

---

## Part 7: Threat hunting — recognizing attack patterns in traffic

| Behavior | What it looks like in the GUI | Display filter |
|---|---|---|
| Command & Control (C2) beaconing | Small, persistent, evenly-spaced TCP connections over time (periodic spikes in **Statistics > I/O Graphs**) | `tls.record.content_type == 23 and frame.len < 300` |
| Domain Generation Algorithms (DGA) | High-entropy, long, randomized DNS queries trying to resolve non-existent domains | `dns.qry.name matches "[0-9a-zA-Z]{16}"` |
| Automated exfiltration | HTTP POST requests from programmatic clients (not real browsers) | `http.request.method == "POST" and http.user_agent contains "python"` |

**The underlying idea I'm taking from this**: malware doesn't usually
announce itself — it hides in traffic *shapes*. Beaconing looks like a
heartbeat because it literally is one (the malware "checking in").
Randomized DNS names look wrong because human-chosen domain names have
patterns and pronounceable structure; DGA output doesn't.

### Hunting inside HTTPS/DNS specifically
Since I usually can't decrypt someone else's traffic, I hunt using
**metadata** instead of content:

```
tls and ip.addr != 192.168.1.0/24              # isolate external TLS traffic
tls.handshake.type == 1                         # view Client Hellos
tls.handshake.extensions_server_name            # extract the target domain (SNI)
```
Then manually check for self-signed certificates, weak ciphers, or
mismatched names — these are red flags even without decrypting the
payload itself.

**Noise reduction for DNS hunting** — strip out the huge volume of normal
background lookups to expose what's actually rare/suspicious:
```
dns and !(dns.qry.name contains "google") and !(dns.qry.name contains "microsoft") and !(dns.qry.name contains "cloudflare")
```

---

## Part 8: TShark — doing all of this without the GUI

`tshark` is Wireshark's command-line version — same engine, scriptable,
and pipeable into normal Unix tools.

**Core operations:**
```bash
tshark -D                          # list available interfaces
sudo tshark -i 1                   # live capture on interface 1
tshark -r file.pcap -w output.pcap # read then write to disk
```

**Filtering:**
```bash
tshark -r file.pcap -f "tcp port 80"                    # capture filter (pre-disk)
tshark -r file.pcap -Y "ip.addr == 192.168.1.100"        # display filter (analysis)
```

**Statistical analysis, no GUI needed:**
```bash
tshark -r file.pcap -z io,phs        # protocol hierarchy
tshark -r file.pcap -z conv,tcp      # conversation matrix
tshark -r file.pcap -z endpoints,ip  # top talkers by endpoint
```

**Chaining into Unix tools — this is the part that clicked for me:**
```bash
tshark -r capture.pcap -Y "http" -T fields -e ip.src | sort | uniq -c
```
Reading this left to right: **intake** (read the file) → **filter node**
(apply the `http` display filter) → **extraction node** (`-T fields -e
ip.src` strips away everything except the source IP) → **Unix
processing** (`sort | uniq -c` counts and ranks who's talking most).
This is genuinely the same "small tools piped together" philosophy from
my log analyzer project, just applied to live network data instead of a
saved log file.

---

## Part 9: Forensic handling of capture files

If a `.pcap` is potential evidence, how I handle the file itself matters
as much as what's inside it:

| Step | Purpose | Tool |
|---|---|---|
| 1. Merge | Combine multiple interface captures into one continuous timeline | `mergecap` |
| 2. Sanitize | Remove corrupted frames or highly sensitive data (passwords) before sharing externally | `editcap` |
| 3. Annotate | Add frame-specific comments inside the file documenting investigative findings | Wireshark GUI (Packet Comment) |
| 4. Seal | Hash the final file to cryptographically prove it hasn't been altered since | `sha256sum` |

**Useful commands:**
```bash
mergecap -w merged.pcapng capture1.pcap capture2.pcap
editcap -t 0 merged.pcapng sorted.pcapng          # ensure chronological order

editcap -c 50000 large.pcap split.pcap            # split by packet count
editcap -i 300 large.pcap split.pcap              # split by time (5 min chunks)
editcap -r raw.pcap clean.pcap 100 250 400        # remove specific frame numbers

sha256sum clean.pcapng > hash.sha256              # seal the evidence
sha256sum -c hash.sha256                          # verify later — proves no tampering
```

**Why this sequence matters, not just the commands**: in an incident
response or legal context, an unsealed, unhashed file has no proof it
wasn't edited after capture. The hash is what turns "trust me" into
"here's cryptographic proof."

---

## Part 10: Full lab walkthrough — capturing a plaintext login (DVWA)

**The scenario**: DVWA (Damn Vulnerable Web App) deliberately runs on
unencrypted HTTP, which makes it a safe, legal way to practice hunting
for plain-text credentials moving across a network — something that
would be illegal to attempt against a real, non-consenting target.

**The setup:**
- Target IP: `192.168.153.23`
- Service: Port 80 (HTTP)
- Capture filter: `host 192.168.153.23 and port 80` (isolating the noise
  before it's even written to disk)
- Trigger: a user submits a login attempt via
  `http://192.168.153.23/dwva/login.php`

**Step-by-step extraction:**
1. **Filter for the action** — `http.request.method == "POST"` finds the
   exact moment data was sent to the server
2. **Locate the target** — identify the specific packet:
   `POST /dvwa/login.php HTTP/1.1`
3. **Reassemble the puzzle** — right-click the packet → **Follow > TCP
   Stream**. This works because TCP segments arrive out of order or in
   pieces (Seq 1, Seq 2, Seq 3...), and Wireshark reconstructs them
   **exactly as the destination server saw them**, giving the complete
   client-server conversation as one readable block instead of scattered
   fragments

**The result** — the reconstructed stream reveals:
```
POST /dvwa/login.php HTTP/1.1
Host: 192.168.153.23
User-Agent: Mozilla/5.0
...
username=admin&password=password&Login=Login
```

Right there in plain text — no decryption needed, because HTTP (unlike
HTTPS) was never encrypted in the first place.

**Preserving this as evidence**: File > Export Specified Packets, save as
`.pcapng`, then seal with `sha256sum` for incident response — this
connects directly back to Part 9's chain-of-custody process.

---

## Final takeaways I want to hold onto
1. **Visibility is control** — without TLS encryption, everything
   (passwords, cookies, session tokens) is exposed in raw packet data.
   This is *why* HTTPS-everywhere matters, seen from the attacker's side
   instead of just being told "encryption is good practice."
2. **Data custody is a real skill, not paperwork** — merge, sanitize,
   annotate, seal. A capture file without this process is not usable
   evidence, no matter how damning its contents look.
3. **The actual capability I've built here**: capture, filter,
   reconstruct, and cryptographically seal a record of what really
   happened on a network. That's a genuinely complete, real skill chain
   — not just "I know how to open Wireshark."

## What I'd still want to practice
- Actually running the TLS decryption workflow myself end-to-end (Part
  6), rather than just reading the steps — I understand the theory but
  haven't executed it hands-on yet
- Getting faster at recognizing the threat-hunting traffic *shapes* from
  Part 7 without needing to look up the filter syntax each time