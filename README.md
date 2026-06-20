# Hardened Network Infrastructure: Internal, Router & External Security Deployment
### *Engineered a hardened MikroTik-based network infrastructure through a layered security architecture spanning Internal, Router, & External. Internal security controls included 802.1X RADIUS authentication, VLAN segmentation, DHCP Snooping, and MACsec to secure endpoint and Layer 2 communications. Router security was reinforced through port knocking, management plane restrictions, custom administrative access controls, and encrypted DNS resolution using DNS over HTTPS (DoH). External security was implemented using a stateful default deny firewall with dynamic port-scan and brute-force detection, Bogon filtering, reconnaissance controls, and multi-vector SYN, UDP, and ICMP flood mitigation to protect against perimeter threats.*

> My Fifth and last project as an unemployed.

---

## The Structure

**Internal Layer (Switching Plane):** 802.1X RADIUS authentication + MACsec encryption + DHCP Snooping + VLAN isolation

**Router Layer (Management Plane):** Port knocking + Custom management ports + DoH + Disabled discovery services

**External Layer (Edge Firewall):** Stateful default-deny + Bogon filtering + Flood mitigation + Brute-force protection

---

## The Lore

Okay so here's the scenario:

I've built all these networks—redundant cores, load-balanced edges, multicast streaming. But one thing kept nagging at me.

**Is any of it actually secure?**

A rogue device plugged into a switch port. A DHCP server hijacking the network. Someone sniffing Layer 2 traffic. An attacker port-scanning the WAN interface. A DDoS flood crashing the edge.

Each layer has its own attack surface. Each needs its own defense.

So for this project, I didn't just add a firewall and call it done. I went **layer by layer**:

- **Internal (Switching Plane):** Who gets on the network? Where do they go? Is their traffic encrypted?
- **Router (Management Plane):** Who can admin the router? How do they get in? What services are exposed?
- **External (Edge Firewall):** What traffic enters from the internet? What gets dropped? What gets blocked?

---

## Internal Security (The Switching Plane)

This layer controls who can plug into the network and what they can reach.

### 802.1X RADIUS Authentication

**What I did:** Configured port-based authentication on all access switches. Any device that plugs in must authenticate against a RADIUS server before being granted network access. No cert? No connection.

**What it stops:** Unauthorized devices. Rogue laptops. Random access points. Someone plugging into a conference room jack.

![802.1X Authentication](https://YOUR-IMAGE-HOST.com/8021x-radius.gif)

---

### VLAN Segmentation & Unused Port Containment

**What I did:** Assigned authenticated users to specific VLANs based on their RADIUS profile. HR goes to HR VLAN. IT goes to IT VLAN. Guests go to guest VLAN. Unused physical ports? Shut down administratively.

**What it stops:** Lateral movement. A compromised HR laptop can't reach IT infrastructure. An unused port can't be exploited.

![VLAN Isolation](https://YOUR-IMAGE-HOST.com/vlan-segmentation.png)

---

### DHCP Snooping

**What I did:** Enabled DHCP Snooping on all access switches. Configured trusted ports (uplinks to core routers) and untrusted ports (access ports facing users). The switch now only accepts DHCP responses from trusted ports.

**What it stops:** Rogue DHCP servers. A malicious user plugging in a laptop running a DHCP server can't hijack the network. DHCP starvation attacks get dropped.

**The proof:** Attempted to spin up a rogue DHCP server on an access port. The switch blocked all DHCPOFFER packets. Zero leases handed out.

![DHCP Snooping](https://YOUR-IMAGE-HOST.com/dhcp-snooping.gif)

---

### MACsec (Layer 2 Encryption)

**What I did:** Enabled MACsec on trunk links between switches and core routers. All Layer 2 traffic between infrastructure devices gets encrypted and integrity-protected at wire speed. Hardware-accelerated. No CPU penalty.

**What it stops:** Someone tapping a fiber link between switches. A compromised switch port mirroring traffic. Any Layer 2 sniffing attack.

**The proof:** Ran Wireshark on a mirrored trunk port. All traffic between switches appeared as encrypted gibberish. No MAC addresses, no VLAN tags, no payloads visible.

![MACsec Encryption](https://YOUR-IMAGE-HOST.com/macsec-wireshark.png)

---

## Router Security (The Management Plane)

This layer controls who can administer the router and how they get in.

### Port Knocking

**What I did:** Configured a sequence of hidden port knocks before SSH becomes visible. Syn packet to port 10001, then 20002, then 30003. Only after the correct sequence does port 22 open for business.

**What it stops:** Automated scanners. Port scanning bots hit closed ports and move on. They never see SSH. They never get a chance to brute-force.

**The proof:** Ran nmap from external host before knocking. Port 22 showed as closed. Ran the knock sequence. Nmap showed port 22 open. Unauthorized scanners see nothing.

![Port Knocking](https://YOUR-IMAGE-HOST.com/port-knocking.gif)

---

### Custom Management Ports

**What I did:** Moved all management services off default ports. SSH on port 56789. WinBox on a non-standard port. API on another random high port. Then locked management access to specific allowed IP subnets.

**What it stops:** Drive-by scans looking for port 22. Automated brute-force tools targeting default management interfaces.

---

### Disabled Unnecessary Services

**What I did:** Shut down everything not explicitly required:
- MAC Telnet Server
- MAC WinBox Server
- MNDP (MikroTik Neighbor Discovery Protocol)
- Bandwidth-test server
- Discovery protocols on public interfaces
- Web interface (HTTP/HTTPS) disabled

**What it stops:** Information leakage. MNDP broadcasts router identity to anyone listening. MAC login bypasses normal authentication. Bandwidth-test can be used for DDoS reflection.

**The proof:** Ran a discovery scan from an external host. No MNDP responses. No MAC server detection. No service banners. Router is silent.

---

### DNS over HTTPS (DoH)

**What I did:** Configured the router to use DNS over HTTPS for all its own lookups. Traffic to 1.1.1.1 and 9.9.9.9 gets encrypted inside HTTPS tunnels. No plaintext DNS leaving the router.

**What it stops:** DNS spoofing. On-path attackers can't see or modify DNS queries. ISP can't log or hijack resolution.

---

## External Security (The Edge Firewall)

This layer controls what traffic enters from the internet.

### Stateful Default-Deny Policy

**What I did:** Built a firewall with an explicit deny at the end of every chain. Input chain? Deny. Forward chain? Deny. Output chain? Allow only necessary.

**The logic:** If it's not explicitly allowed, it's blocked. No implicit permits. No "any any" rules.

---

### Port Scan Detection & Blocking

**What I did:** Configured PSDB (Port Scan Detection Blocking) in MikroTik. Tracked connection attempts. When a single source hits too many ports within the time threshold, the firewall adds them to a dynamic block list.

**The proof:** Ran nmap -p 1-1000 from an external host. After 15 ports, the firewall flagged the scan and blocked the source IP for 10 minutes. The scan stopped receiving any responses.

![Port Scan Block](https://YOUR-IMAGE-HOST.com/port-scan-block.gif)

---

### Brute-Force Attack Prevention

**What I did:** Set up connection tracking for SSH and WinBox management ports. If a source IP makes more than 5 failed login attempts in 30 seconds, the firewall dynamically blocks them for 1 hour.

**The proof:** Simulated brute-force attempt with Hydra. After 5 failed attempts, the firewall added the source to address-list "bruteforce_blocked". Subsequent attempts got connection refused.

---

### Bogon IP Filtering

**What I did:** Added bogon filter rules at the top of the input and forward chains. Blocked reserved IP ranges like 0.0.0.0/8, 127.0.0.0/8, 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16, 224.0.0.0/4, 240.0.0.0/4.

**What it stops:** Spoofed source IPs. Traffic claiming to come from internal addresses arriving on external interfaces. Martian packets.

---

### SYN, UDP & ICMP Flood Mitigation

**What I did:** Configured rate-limiting on SYN packets, UDP traffic, and ICMP messages. Set thresholds based on normal traffic patterns. Applied to raw table to offload CPU.

**The proof:** Generated a SYN flood from an external host. MikroTik's raw table filtering dropped the excess packets before they hit connection tracking. CPU stayed below 15% during a flood that would have pegged it at 100%.

![Flood Mitigation](https://YOUR-IMAGE-HOST.com/flood-mitigation.gif)

---

### ICMP Smurf Attack Mitigation

**What I did:** Blocked ICMP echo requests destined to broadcast addresses. Configured the firewall to drop ICMP packets where destination is 255.255.255.255 or subnet broadcast.

**What it stops:** Smurf attacks where attackers spoof a victim's IP and ping a broadcast address, causing every host on the network to reply and flood the victim.

---

### ICMP Policy Control & Traceroute Limitation

**What I did:** Allowed only essential ICMP types:
- Echo reply (0)
- Echo request (8) - rate-limited
- Destination unreachable (3)
- Time exceeded (11) - for traceroute

Blocked everything else. Additionally, limited traceroute by rate-limiting time-exceeded responses.

**What it stops:** Network reconnaissance. Attackers use traceroute to map the network. Now they get incomplete results.

---

## Security Layer Summary

| Layer | Protections Deployed | Primary Threat Mitigated |
|-------|---------------------|--------------------------|
| **Internal (Switching)** | 802.1X, VLAN isolation, DHCP Snooping, MACsec | Unauthorized access, rogue DHCP, Layer 2 sniffing |
| **Router (Management)** | Port knocking, custom ports, disabled services, DoH | Automated scans, brute-force, info leakage |
| **External (Firewall)** | Default-deny, bogon filtering, flood mitigation, brute-force blocking | DDoS, port scans, reconnaissance, spoofing |

---

## Challenges I Faced

**Port Knocking and State Table Exhaustion**

During initial port knocking testing, I realized that each knock packet created connection state entries. An attacker could trigger thousands of knocks and fill the connection table.

**How I solved it:** Moved the knock detection to the raw table. Connection tracking never sees the knock packets. No state table impact.

**MACsec Key Negotiation**

Getting MACsec to work between different switch chips required careful coordination. The keys wouldn't negotiate. Switches kept dropping back to unencrypted.

**How I solved it:** Verified that both ends supported the same MACsec cipher suite (GCM-AES-128). Ensured MKA timers were synchronized. Pre-shared keys configured identically on both sides.

**DoH and Local DNS Resolution**

Enabling DoH broke local DNS resolution for internal devices. The router was sending internal hostnames to Cloudflare.

**How I solved it:** Configured split DNS. Internal domains resolved locally via the router's cache. External domains went through DoH. Internal clients never leaked internal hostnames.

---

## Final Thoughts

Security isn't one thing. It's a stack.

A firewall alone won't stop someone plugging into a conference room jack. MACsec won't stop someone brute-forcing SSH. Port knocking won't stop a rogue DHCP server.

You need every layer defending its own attack surface.

**The Metric That Matters: Defense in Depth**

I measured this by simulating attacks at every layer:
- Rogue DHCP server? Blocked at the switch.
- Unauthorized device? 802.1X denied it.
- Port scan? Dynamic block kicked in after 15 ports.
- SYN flood? Raw table dropped the excess.
- Brute-force? Blocked after 5 attempts.

Each attack hit a wall. Each layer did its job.

---

*Last project for now.*
