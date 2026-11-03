# Design Decisions

Why the network is built the way it is, including the choices forced by Packet Tracer. For how each decision would change in production, see [Lab vs Production](../README.md#lab-vs-production) in the README.

## Scope

| Decision | Reason |
|---|---|
| Two sites: a two-floor HQ and a Sales branch | Covers campus design, WAN, and branch connectivity without duplicating near-identical branch builds. An earlier three-site plan was cut to keep the lab stable and the documentation focused. |
| Departments on separate VLANs (IT, HR, Finance, Servers, Voice, Guest, Management) | Lets each department be filtered by ACL and keeps broadcast domains small. |
| Wireless in a separate lab file | The full topology plus a WLC, access points and wireless clients made Packet Tracer 9.0 lag and crash ([TKT-011](troubleshooting-log.md#tkt-011-packet-tracer-lag-and-crashes-with-wireless-added)). A wired guest kiosk in VLAN 70 keeps guest isolation testable in the main lab. |

## Campus

| Decision | Reason |
|---|---|
| Two-tier collapsed core instead of three tiers | A site with two floors and three access switches does not need a separate distribution layer. Two Layer 3 core switches give the same redundancy with fewer devices. |
| Two Catalyst 3650 cores joined by EtherChannel | Either core can fail without losing the site, and the inter-core link has two members. |
| HSRPv2 with active roles split across the cores | Both cores forward traffic in normal operation instead of one sitting idle. Preemption returns each VLAN to its preferred core after a failure. |
| STP root matched to the HSRP active core per VLAN | Keeps Layer 2 and Layer 3 paths aligned so traffic does not cross the core-to-core link just to reach its gateway. |
| Static EtherChannel (`mode on`) | LACP was unstable in Packet Tracer ([TKT-010](troubleshooting-log.md#tkt-010--etherchannel-flapping-hsrp-active-on-both-cores)). |
| Native VLAN 999, static trunks, VTP transparent | Removes DTP negotiation and VTP database risk, and keeps untagged traffic off user VLANs. |
| Copper uplinks between floors | The Packet Tracer 2960 has no fiber ports. |

## Routing and WAN

| Decision | Reason |
|---|---|
| ISR 2911 routers instead of ISR 4331 | The 4331 in this Packet Tracer version only has two usable ports, not enough for LAN, WAN, ISP and cross-link connections. |
| Two WAN carriers to the branch | Removes the WAN link as a single point of failure for the branch. |
| OSPF between the edges and the branch, with cost 10 on Carrier-A and 100 on Carrier-B | Makes Carrier-A the primary path and Carrier-B the automatic backup. |
| `distance 111 2.2.2.2` on the branch router | Packet Tracer rejected the usual fixes (`metric`, `metric-type 1`, `maximum-paths`), and equal-cost defaults broke NAT sessions ([TKT-014](troubleshooting-log.md#tkt-014--branch-web-pages-failed-while-pings-worked)). |
| Serial cross-link between the edge routers | Gives each edge a path to every site, and keeps branch-to-HQ traffic symmetric through the firewall during a WAN failover ([TKT-016](troubleshooting-log.md#tkt-016--branch-to-hq-traffic-did-not-fail-over-to-carrier-b)). |
| Centralized internet breakout at HQ | The branch reaches the internet through HQ, so all internet traffic passes the HQ firewall and edge NAT. |

## Security

| Decision | Reason |
|---|---|
| ASA between the core and the edge, with two inside and two outside interfaces | A single edge router or core switch can fail and a path through the firewall remains. |
| Static routing on the ASA | Packet Tracer's ASA accepted OSPF config but did not form adjacencies. |
| Inbound ACLs on the HR, Finance and Guest SVIs, identical on both cores | Filters traffic closest to the source, and the policy still holds when the other core takes over. |
| Port security, DHCP snooping, DAI, PortFast and BPDU guard on access ports | Covers the common Layer 2 attacks: rogue devices, rogue DHCP servers, ARP spoofing and accidental loops. |
| SSH only, limited to the management VLAN | Management access requires local credentials and must come from 10.1.99.0/24. |
| Unused ports shut down and placed in VLAN 999 | Nothing can be plugged in and used without a config change. |

## Services

| Decision | Reason |
|---|---|
| One DHCP server for both sites, reached through relay | One place to manage every pool. The branch reaches it across the WAN. |
| Central syslog and NTP on HQ-LOG-SRV-01 | Logs from every device land in one place with matching timestamps. |

## Naming

Hostnames follow `SITE-LOCATION-ROLE-NUMBER`, so a name tells you where a device is and what it does. The location part is left out when a device serves the whole site.

| Part | Values | Example |
|---|---|---|
| SITE | `HQ`, `BR1` | `BR1-RTR-01` |
| LOCATION | `F1`, `F2` (floor) for HQ access switches, phones and printers, omitted for site-wide devices and PCs | `HQ-F2-ASW-01`, `HQ-F2-PRN-01` |
| ROLE (network) | `CORE`, `ASW` (access switch), `SRV-SW`, `EDGE`, `FW`, `RTR` | `HQ-CORE-01` |
| ROLE (endpoint) | Department plus type: `IT-PC`, `IT-ADMIN`, `HR-PC`, `FIN-PC`, `SALES-PC`, `PH` (phone), `PRN` (printer), `GUEST-KIOSK` | `HQ-HR-PC-02` |
| ROLE (server) | Service plus `SRV`: `DHCP-SRV`, `DNS-SRV`, `WEB-SRV`, `LOG-SRV` | `HQ-LOG-SRV-01` |
| NUMBER | Two digits | `01` |
