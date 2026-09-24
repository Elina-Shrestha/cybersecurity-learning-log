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
