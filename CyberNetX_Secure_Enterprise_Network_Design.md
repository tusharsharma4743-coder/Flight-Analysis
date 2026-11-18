# Design and Implementation of Secure Enterprise Infrastructure
### Cisco Packet Tracer (CPT) Network Design — CybernetX Project

---

## 1. Project Scope

This topology models a mid-sized enterprise (HQ + Branch + DMZ) with:
- Three-tier hierarchical design (Core / Distribution / Access)
- VLAN segmentation per department
- Perimeter security (firewall, ACLs, zone-based policies)
- A public-facing DMZ (Web, DNS, Mail)
- Site-to-site VPN between HQ and Branch
- AAA authentication, port security, and SSH-only management
- Gateway redundancy (HSRP)
- Wireless access with WPA2-Enterprise
- Traffic capture points for Wireshark analysis

---

## 2. Logical Topology (Text Diagram)

```
                              [ISP Cloud (Router-ISP)]
                                |                |
                     [HQ Edge Router R1]   [Branch Edge Router R2]
                        |    (VPN Tunnel: IPsec over GRE)   |
                        |                                    |
                 [ASA 5505 Firewall]                  [Branch L3 Switch]
                        |                                    |
        -------------------------------                [Branch VLANs:
        |                             |                  Sales-Branch,
   [DMZ Switch]                [Core Switch - L3]         Mgmt-Branch]
   |    |    |                  /        |        \             |
  Web  DNS  Mail          Dist-SW1   Dist-SW2   Dist-SW3   [Access SW]
  Srv  Srv  Srv              |          |           |           |
                          Access-SW  Access-SW   Access-SW   [PCs]
                             |          |           |
                          VLAN10     VLAN20      VLAN30
                          (HR)      (Finance)    (Sales)
                             |          |           |
                          PCs/Laptops PCs/Laptops PCs/Laptops

        Additional at Core layer:
        - VLAN 40 (IT/Admin)
        - VLAN 50 (Management - Syslog, AAA/RADIUS, NTP)
        - VLAN 99 (Native/Unused - security best practice)
        - WLC + Access Point serving VLAN 60 (Wireless)
```

---

## 3. Device List (Packet Tracer components)

| Role | Device (PT model) | Qty |
|---|---|---|
| ISP simulation | Generic Router (1941/2911) | 1 |
| HQ Edge Router | Router 2911 | 1 |
| Branch Edge Router | Router 2911 | 1 |
| Firewall | ASA 5505 | 1 |
| Core Switch (L3) | Switch 3560/3650 (multilayer) | 1 |
| Distribution Switches | Switch 2960/3560 | 2–3 |
| Access Switches | Switch 2960 | 3–4 |
| DMZ Switch | Switch 2960 | 1 |
| Servers | Server-PT (Web, DNS, Mail, DHCP, AAA/RADIUS, Syslog) | 6 |
| Wireless LAN Controller | WLC 2504 | 1 |
| Access Point | AP-PT-N (Lightweight) | 1–2 |
| End Devices | PC-PT, Laptop-PT | 12–20 |
| Branch L3 Switch | Switch 3560 | 1 |

---

## 4. IP Addressing Scheme

| Segment | VLAN | Network | Gateway |
|---|---|---|---|
| HR | 10 | 192.168.10.0/24 | 192.168.10.1 |
| Finance | 20 | 192.168.20.0/24 | 192.168.20.1 |
| Sales | 30 | 192.168.30.0/24 | 192.168.30.1 |
| IT/Admin | 40 | 192.168.40.0/24 | 192.168.40.1 |
| Management | 50 | 192.168.50.0/24 | 192.168.50.1 |
| Wireless | 60 | 192.168.60.0/24 | 192.168.60.1 |
| Native (unused) | 99 | 192.168.99.0/24 | — |
| DMZ | — | 172.16.10.0/28 | 172.16.10.1 |
| WAN (R1–ASA) | — | 10.0.0.0/30 | — |
| WAN (R1–ISP) | — | 203.0.113.0/30 | — |
| WAN (R2–ISP) | — | 203.0.113.4/30 | — |
| Branch LAN | 70 | 192.168.70.0/24 | 192.168.70.1 |
| Site-to-Site VPN tunnel | — | 172.16.20.0/30 | — |

Use **VLSM** and show your subnetting table in the report — examiners usually check this closely.

---

## 5. Core Configuration Elements to Implement

### A. Switching Layer
- Trunk links between core/distribution/access (802.1Q), native VLAN moved to 99
- Inter-VLAN routing via SVIs on the L3 core switch (or router-on-a-stick as a fallback if only L2 switches are available)
- **Port security** on access ports: max MAC 2, violation = restrict, sticky MAC
- **DHCP snooping** and **Dynamic ARP Inspection** on access switches (if using 3560/2960 with IOS supporting it)
- **Spanning-Tree**: PVST+, root bridge fixed on core switch, BPDU Guard on access ports

### B. Routing
- OSPF (or EIGRP) between R1, R2, Core L3 switch, Branch L3 switch — single area is fine for a project
- Route summarization at HQ/Branch boundary
- Default route from R1 to ISP

### C. Firewall / Perimeter (ASA 5505)
- Interfaces: Outside (low security 0), Inside (security 100), DMZ (security 50)
- NAT/PAT: inside → outside PAT using ASA outside IP
- Static NAT for DMZ servers (Web/Mail) so they're reachable from outside
- ACLs: only allow TCP 80/443 to Web server, TCP 25/110 to Mail, UDP 53 to DNS from outside; deny everything else inbound

### D. Zone-Based / ACL Security on Routers
- Extended ACLs on R1 blocking unwanted inbound traffic before it even hits the ASA
- Standard ACLs restricting Telnet/SSH management access to the Management VLAN only

### E. VPN
- Site-to-site IPsec VPN (GRE tunnel encrypted with IPsec, or native IPsec VPN) between R1 (HQ) and R2 (Branch)
- Pre-shared key authentication, ISAKMP phase 1/2 policy shown in report screenshots

### F. AAA / Authentication
- RADIUS or TACACS+ server (Server-PT) for centralized login on switches/routers
- Local fallback account configured
- `line vty` restricted to SSH only, version 2, with domain name + crypto key generated

### G. Wireless Security
- WLC + AP broadcasting SSID mapped to VLAN 60
- WPA2-Enterprise (802.1x with RADIUS) — or WPA2-PSK if RADIUS integration isn't supported cleanly in your PT version
- Disable SSID broadcast optionally, for defense-in-depth discussion

### H. Logging / Monitoring
- Syslog server (Server-PT) receiving logs from switches/routers/ASA
- NTP server for time sync (important for correlating logs)

---

## 6. Wireshark Integration Plan

Since Packet Tracer's "Simulation Mode" shows PDUs but you'll pair this with **real Wireshark captures** conceptually (or PT's own packet inspection), structure your report around these capture points:

| Capture Point | What to Show |
|---|---|
| Between PC (HR VLAN) and Core Switch | Normal ARP + DHCP request/response, VLAN tag if trunked |
| Between R1 and ASA | NAT translation before/after (PAT changing source IP/port) |
| Between ASA and DMZ | HTTP/DNS/SMTP request reaching servers, blocked traffic dropped |
| VPN tunnel between R1 and R2 | Encrypted ESP packets (payload unreadable) vs plaintext without VPN |
| Failed login attempt on SSH | TCP handshake but no cleartext credentials (contrast with insecure Telnet capture to prove why SSH matters) |
| Port security violation | Switch logs / err-disable triggered by MAC flooding attempt |

This lets you build a "before hardening vs after hardening" comparison — a strong angle for the report: e.g., capture Telnet in cleartext first, then show SSH encrypted equivalent.

---

## 7. Redundancy (Optional but Strong Addition)
- **HSRP** or **VRRP** between two core/distribution switches for default gateway redundancy
- Dual uplinks with EtherChannel (LACP) between access and distribution layers
- Show failover test: shut the active gateway interface, verify standby takes over, capture the convergence time

---

## 8. Suggested Report Structure (mapping design → deliverables)

1. Introduction & objectives
2. Network requirements analysis
3. Logical & physical topology diagrams (export from PT)
4. IP addressing & VLAN table
5. Device configurations (paste running-configs, screenshots)
6. Security implementation (ACLs, firewall rules, AAA, port security, VPN)
7. Testing & verification (ping, traceroute, show commands)
8. Wireshark packet analysis (before/after hardening comparisons)
9. Challenges faced & conclusion

---

## 9. Build Order (practical steps in Packet Tracer)

1. Place all devices, cable physical topology first
2. Configure VLANs + trunks + access ports
3. Configure inter-VLAN routing (SVIs or router-on-a-stick)
4. Configure OSPF between all L3 devices
5. Configure ASA firewall interfaces, NAT, ACLs
6. Bring up DMZ servers, verify from outside via static NAT
7. Configure site-to-site VPN
8. Configure AAA/RADIUS + SSH-only management
9. Configure port security + DHCP snooping
10. Add wireless (WLC + AP), test WPA2-Enterprise
11. Add HSRP for redundancy
12. Test end-to-end connectivity, then begin failure/attack simulations for your Wireshark screenshots

---

*This design deliberately includes more security layers than a minimal project would need — trim any section (e.g., HSRP, wireless, VPN) if your course scope or time is limited, but keep VLANs, ACLs, firewall, and AAA as the non-negotiable core for a "secure enterprise infrastructure" thesis.*
