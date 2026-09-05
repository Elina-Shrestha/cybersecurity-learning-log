# Networking

**One analogy for the whole module**
- Payload; The letter: Actual data/message
- Header; The envelope: Addressing for routing
- IP/Port; The address: Which building, which flat, two different levels of address
- Frame; The post van: Physical wire/Radio waves that carry it in one hop
- Router; The sorting office: Reads the address, decides the next hop, without reading the data inside
- Firewall; The X-ray machine: Inspects the data against rules and blocks them

**Terms to know in network**
- Endpoints: Devices that want to communicate
- Medium: Physical wire/Radio waves; through what means communication takes place
- Protocol: Rules that need to be followed during communication
- PAN: Personal Area Network; phone and earbuds
- LAN: Local Area Network; same building
- WAN: Wide Area Network; city to city

*Everything in a network travels in bits*

# Devices in the middle

- Hub: Repeats every signal out of every port. Has no idea about MAC address. Everyone sees everyone's traffic. Open wifi works this way.
- Switch: Learns what MAC address live on which port, forwards frame only where they need to go. Better isolation but can be easily flooded/ lied to into sending you someone else's packets
- Router: Joins networks, reads destination IP, consults routing table, decrements TTL and rebuilds the frame. Natural chokepoint for networks thats why firewall generally live here.

*Switch moves frame within one network and Router between networks.*

# OSI Model

7 layers reference model. It is called a reference model before it is just used as a look-up model as it is more detailed. Otherwise TCP/IP model is widely used.
|Layer No.|Layer Name|What it is responsible for|Examples|Attacks that live here|
-----------------------------------------------------
|7|Application Layer|Protocol that you device speaks|http,smtp,ftp,dns|Phishing,SQL injection|
|6|Presentation Layer|Formatting,encoding,encryption|ASCII,UTF-8,JPEG|Weak cipher,expired certs|
|5|Session Layer|Start, maintain and end a dialogue|TLS session resumption|Session hijacking,token theft|
|4|Transport Layer|Reliability,ordering and port numbers|TCP,UDP|SYN flood,port scanning|
|3|Network Layer|Logical Addressing and routing|Router,IP,ICMP|IP spoofing,ICMP tunneling|
|2|Data Layer|Getting frame in one physical hop|MAC,switches,ethernet|MAC flooding,ARP spoofing|
|1|Physical Layer|Raw Signals-radio,light,voltage|Cable,fibre,WiFi signals|Jamming,theft|

# Layers that actually do the delivery

1. Physical- Turns bits into signal and back. Failure looks like: dead port, dodgy cable, no link light
2. Data- Delivers frames to devices within the same network. Failure looks like: link light is up but no traffic, duplicate MAC
3. Network- Finds paths across many networks. Failure looks like: can ping the gateway but cannot ping the internet
4. Transport- Delivers to the right program, reliably or not. Failure looks like: host is up but port is dead

# TCP/IP Model

|Layers|Maps onto OSI|What runs here|
-------------------------------------
|Application Layer|Session, Presentation, Application|HTTP,SMTP,TLS,DNS|
|Transport Layer|Transport|TCP,UDP|
| Internet Layer|Network|IP,ICMP|
|Network Access Layer|Physical, Data|Ethernet, WiFi, Cables|

# TCP VS UDP

|TCP|UDP|
---------
|Connection-oriented-Handshake happens first|Connection-less-No handshake|
|Every byte is numbered and must be acknowledged|No ordering, acknowledgement or retransmission|
|Lost packets are retransmitted|Less overhead- 8 byte header instead of 20|
|Slower during network conjestion|Faster|
|Costly due to extra round trips, header|Tivially spoofable- Sender is not verified|
|Data is received by the app in right order|Data is not received in correct order|

# TCP- Three way Handshake

1. SYN: Client-> Server: I want to talk, my starting sequence number is A.
2. SYN-ACK: Server-> Client: Fine. I acknowledge A, my starting sequence number is B.
3. ACK: Client-> Server: I acknowledge B, connection established.
Attackers abuse every one of them.
- SYN flooding: Sends multiple syn message but never gets a ack back. Server holds each half opened connection in memory until it runs out.
- SYN scan: Sends a SYN request and gets a RST instead of ACK. You learn whether the port is open without completing the connection.