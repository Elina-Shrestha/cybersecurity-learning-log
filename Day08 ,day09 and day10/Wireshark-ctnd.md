Reading the Wire
Protocols, addressing, translation and access control — the vocabulary you need before Wireshark

Theory session · No lab required · Prerequisite for Module 04: Packet Analysis with Wireshark

### Notes:
TIMING: 2 min.
OPEN WITH: 'Next week you will open Wireshark and see ten thousand lines scroll past. Today is the session that decides whether that looks like noise or like sentences.'
SET EXPECTATION: this is a pure theory session. No VM, no commands. Notebooks out.
STUCK POINT: students who wanted the tool immediately will disengage. Tell them explicitly: every term today appears in the Wireshark packet detail pane next week. This is the glossary for that.

<!-- Slide number: 2 -->
WHY THIS SESSION EXISTS
Wireshark shows you everything. That is the problem.

What students see
What is actually there
What today fixes

Thousands of rows scrolling past in seconds
Colour bands nobody explained
Words like SYN, TTL, EtherType, RCODE
Three different addresses on one packet
Conclusion: 'this tool is broken or I am stupid'
A conversation, in a strict grammar
Every layer wrapping the one above it
Every field with a defined meaning and size
A small number of protocols doing 95% of the work
Repetition — the same shapes, over and over
You will know the name of every field before you meet it
You will know which layer to look at for which question
You will know why your VM sees what it sees
You will know why some traffic never arrives at all

KEY POINT   Wireshark is a dictionary, not a teacher. It defines nothing you do not already have a word for.

### Notes:
TIMING: 4 min.
ASK THE ROOM: 'Has anyone opened Wireshark before? What did it look like?' Expect 'confusing'. Use that answer.
ANALOGY TO USE: reading a foreign newspaper. The letters are visible, the meaning is not. Today is the alphabet and the grammar.
STUCK POINT: do not let them think the goal is memorising field names. The goal is knowing which layer answers which question.

<!-- Slide number: 3 -->
SESSION OUTCOMES
By the end of this session you can answer these

Structure and addressing
Reachability and control

Which four addresses identify one conversation, and at which layer each lives
Why a packet has both a MAC address and an IP address, and which one changes on the way
What is inside an Ethernet frame, an IP header, a TCP segment and a UDP datagram
What ARP is doing on a network you thought was quiet
What NAT rewrites, and why the same conversation looks different from two capture points
What a bridge does, and which VM network mode lets you capture what
How a name becomes an address, and every record type involved
Why a packet you sent never produced a reply — and how to tell drop from reject

KEY POINT   Write these eight questions in your notebook now. If any is still unclear at the end, ask before you leave.

### Notes:
TIMING: 3 min.
DO THIS: make them physically write the eight questions down. It gives them a checklist to self-assess against and it gives you a closing activity.
CUT GUIDANCE: if you are short on time this slide can be read aloud in 60 seconds rather than discussed.

<!-- Slide number: 4 -->

PART 1
IN THIS PART
The Layered Model
The OSI and TCP/IP models
Encapsulation and decapsulation
Frame, packet, segment, datagram
Headers, payloads and trailers
One web request, end to end
Why one message is wrapped in four envelopes, and what each envelope is for

### Notes:
TIMING: 30 seconds. Divider slides exist so students can hear the gear change. Say the part number out loud.

<!-- Slide number: 5 -->
THE REFERENCE MODEL
OSI: seven layers, and what each one is responsible for
| Layer | Name | Job in one sentence | You will see |
| --- | --- | --- | --- |
| 7 | Application | The thing the user actually wanted — a page, a mail, a file | HTTP, DNS, SMB, FTP |
| 6 | Presentation | Format, encode, encrypt so both ends agree on representation | TLS, character encoding |
| 5 | Session | Open, maintain and close a dialogue between two applications | RPC, NetBIOS sessions |
| 4 | Transport | Deliver to the right program on the host; reliability if asked for | TCP, UDP, port numbers |
| 3 | Network | Deliver across networks, from any host to any host, worldwide | IPv4, IPv6, ICMP, routing |
| 2 | Data Link | Deliver across one physical hop, on one local segment | Ethernet, MAC, ARP, VLAN |
| 1 | Physical | Bits as electricity, light or radio on the medium | Cable, Wi-Fi, signal levels |

KEY POINT   Layers 5, 6 and 7 are blurred in practice. Wireshark labels most of that region simply 'Application'.

### Notes:
TIMING: 6 min.

# Module 04 — Wireshark & Packet Analysis

> **Purpose:** Learn to capture, filter, reconstruct, and interpret network traffic with Wireshark, including what can and cannot be learned from encrypted traffic.

## Module Overview

This module covers two sessions (about four hours total) using Kali Linux and Windows, with four supplied lab captures.

### Learning outcomes

By the end of the module, you should be able to:

1. Capture traffic and inspect packets layer by layer down to individual bytes.
2. Create display filters that isolate the traffic relevant to an investigation.
3. Reconstruct TCP/UDP conversations, follow streams, and recover transferred files.
4. Identify useful information that remains visible even when application data is encrypted.
5. Decrypt an authorised TLS session when the appropriate key log is available.
6. Recognise common signs of abnormal activity such as port scans, periodic beacons, and protocols operating on unexpected ports.

---

# 1. What Wireshark Is

Wireshark is a free, open-source **network protocol analyser**. It captures frames received by a network interface, decodes the protocols it recognises, and presents the decoded information as a navigable protocol tree.

### Important terminology

- **Dissector:** Code used by Wireshark to interpret a particular protocol.
- **libpcap / Npcap:** Capture libraries/drivers that provide frames to Wireshark.
- **pcap / pcapng:** Packet-capture file formats. `pcapng` is the modern default and stores additional metadata.
- **tshark:** Wireshark's command-line counterpart. It uses the same protocol dissectors and filter concepts.

### What Wireshark does not do

Wireshark is an observation and analysis tool, not an enforcement tool.

It does **not**:

- block traffic,
- modify traffic,
- inject traffic,
- automatically break encryption,
- or act as a firewall/IPS.

A useful mental model is: **Wireshark is a microscope for network traffic.**

---

# 2. Common Wireshark Misconceptions

| Misconception | Reality |
|---|---|
| Wireshark hacks into systems | It is primarily passive and analyses frames received by the interface. |
| Promiscuous mode lets you see everybody's traffic | On a switched network, it does not magically deliver other hosts' unicast traffic to your port. |
| Wireshark breaks encryption | Decryption requires legitimate access to the relevant keys. |
| Wireshark blocks attacks | It has no traffic-enforcement capability. |
| A packet labelled HTTP must actually be HTTP | Protocol identification can rely on ports and heuristics and can therefore be wrong. |

---

# 3. How Packet Capture Works

A simplified capture path is:

1. A frame arrives at the physical/network interface.
2. The network card determines whether the destination is its own MAC address, a broadcast, or a subscribed multicast address.
3. Accepted frames are passed to the operating system.
4. The capture mechanism obtains a copy for Wireshark.

### Promiscuous mode

Promiscuous mode changes the interface's filtering behaviour so it accepts every frame that physically reaches it, regardless of destination MAC.

However, this is important:

> **Promiscuous mode cannot make a switched network send traffic to your port.**

If the frames never reach your interface, promiscuous mode has nothing to capture.

### Interface modes

| Mode | What is accepted | Typical visibility |
|---|---|---|
| Normal | Frames for the host, broadcasts, and joined multicast | Mostly your traffic and broadcasts |
| Promiscuous | All frames physically arriving at the interface | Everything reaching that port |
| Monitor mode | Raw 802.11 wireless frames | Wireless traffic within radio range; encrypted payloads remain encrypted |

Monitor mode is specific to Wi-Fi hardware/driver support. Some adapters do not support it, and Windows behaviour can vary.

---

# 4. Hub, Switch, and Ways to Obtain Network Traffic

### Hub

An old-style hub repeats frames out of every port, so every connected host receives the traffic.

### Switch

A switch learns MAC-to-port mappings and normally forwards frames only where they need to go. Therefore, an endpoint capture usually contains that host's own traffic plus broadcasts.

### How network monitoring is normally performed

- **SPAN / port mirroring:** The switch copies traffic to a designated monitoring port.
- **Network TAP:** Hardware placed into a link provides a monitoring copy and is commonly used for evidence-quality capture.
- **Endpoint capture:** Capture directly on the machine under investigation.
- **Gateway capture:** Capture at a router/gateway through which traffic passes.

ARP spoofing can also redirect traffic, but it is an attack technique and should not be performed outside an authorised environment. In this module, the relevant goal is recognising its effects in a capture.

---

# 5. Capture Vocabulary

| Term | Meaning |
|---|---|
| **Snap length** | Maximum number of bytes recorded for each frame. |
| **Ring buffer** | Multiple rotating capture files where the oldest file is overwritten when the buffer fills. |
| **Dropped packets** | Frames the capture process failed to record because it could not keep up. |
| **Timestamp** | Time at which the capture point observed the frame. |
| **Capture point** | The location in the network topology where traffic was captured. |

Always document the capture point. Conclusions from a capture are relative to where that capture was taken.

Dropped packets are particularly important when interpreting TCP-analysis indicators because a capture loss can resemble actual network loss.

---

# 6. Legal, Ethical, and Consent Requirements

Only capture traffic on:

- a network you own, or
- a network for which you have explicit permission from the owner.

For the classroom:

1. Use the authorised lab/VM network.
2. Do not capture campus, hostel, employer, friend's, family, or public Wi-Fi.
3. If credentials or personal information are captured accidentally, delete the material and report it to the instructor.
4. The purpose of packet analysis is to understand, troubleshoot, and defend systems—not to misuse captured information.

### Nepal legal context in the source material

The slides identify two relevant Nepal instruments:

- **Electronic Transactions Act, 2063 (2008):** Includes offences relating to unauthorised access to computer systems/material and unauthorised disclosure of data obtained through such access. Section 44 is identified in the material as the commonly cited provision for unauthorised access.
- **Individual Privacy Act, 2075 (2018):** Addresses personal information and privacy of communications, including restrictions around collecting or disclosing personal data without consent.

The source explicitly advises checking the current consolidated statutory text before relying on the numbering for legal or examination purposes.

---

# 7. Installing Wireshark

## Kali Linux

Wireshark is included with Kali, but capture permissions need to be configured correctly.

```bash
wireshark --version
sudo apt update && sudo apt install -y wireshark tshark

sudo dpkg-reconfigure wireshark-common
sudo usermod -aG wireshark $USER
newgrp wireshark
id | grep wireshark
```

When prompted by `dpkg-reconfigure`, allow non-root packet capture.

The group membership change may require logging out and back in. Avoid running the graphical Wireshark application as root; configure the capture permissions instead.

## Windows

1. Download the Windows x64 installer from the official Wireshark site.
2. Accept/install **Npcap**, which provides packet capture support.
3. Leave raw 802.11 support disabled if it could interfere with normal networking.
4. Reboot if the Npcap installation requests it.

If no capture interfaces appear on Windows, Npcap is a primary thing to check.
---

# 8. Useful Initial Interface Settings

Recommended projector/class settings:

- **Time:** Seconds Since Beginning of Capture
- **Precision:** Milliseconds
- **Zoom:** Increase the packet-list display
- **Network Name Resolution:** Disable it
- **Colorising:** Keep packet colouring enabled

Disabling name resolution avoids unnecessary reverse-DNS lookups from the analysis machine.

---

# 9. Wireshark Interface

## Capture interfaces

Common interfaces include:

- `eth0` — Ethernet
- `wlan0` — Wi-Fi
- `lo` — loopback
- `any` — all interfaces on Linux

The small traffic graph beside an interface helps identify which interface is actually carrying traffic.

### Capture filter

A **capture filter** is applied before packets are recorded and uses BPF/tcpdump-style syntax.

Example:

```text
tcp port 80
```

The loopback interface is useful for safe local captures. The supplied TLS exercise was built using loopback traffic, a local TLS server, and a client-generated key log.

---

# 10. The Three Wireshark Panes

### 1. Packet List

The upper pane contains one row per frame. It is primarily used for finding and scanning packets.

### 2. Packet Details

The middle pane shows the selected frame as a decoded, expandable protocol tree. This is where most analysis takes place.

### 3. Packet Bytes

The lower pane shows the raw bytes in hexadecimal alongside ASCII representation.

A useful distinction:

> Packet List = index, Packet Details = interpretation, Packet Bytes = raw evidence.

If the decoded interpretation and expectations disagree, the raw bytes are the final reference for what was actually present in the capture.

---

# 11. Linked Selection and Headers

Selecting a field in Packet Details highlights the corresponding bytes in Packet Bytes.

For example:

1. Expand IPv4.
2. Select **Time to Live (TTL)**.
3. Observe that one byte is highlighted in the hexadecimal view.

This connects abstract protocol-header diagrams to the actual bytes in a captured frame.

---

# 12. Packet List Columns

| Column | Meaning | Important caveat |
|---|---|---|
| No. | Frame number in this capture file | Not a network-level value |
| Time | Time relative to capture start/configuration | Depends on display settings |
| Source / Destination | Address information shown by Wireshark | The relevant address layer can differ |
| Protocol | Highest protocol identified by dissectors | Identification is not infallible |
| Length | Captured frame length | Truncation can affect what is represented |
| Info | Wireshark-generated summary | Format varies by protocol |

You can create custom columns by right-clicking a field and choosing **Apply as Column**.

Useful custom fields include TCP window size and TLS server name.

---

