# Consumer Edge IPv6 Multihoming (Failover, Load‑Share, and Alternatives)

## Scope & Purpose

This document provides guidance for network engineers and advanced users configuring consumer or SMB edge routers with multiple upstream IPv6 connections.

While IPv4 multihoming is often trivialized by NAT (where the internal network is hidden behind a single public IP per uplink), IPv6 introduces complexity because it prioritizes end-to-end address transparency. In a typical consumer setup, every host on the network receives public IPv6 addresses routed from each ISP. Managing traffic flow, failover, and source address selection without breaking connectivity requires specific router behaviors that are not yet standard in many commercial products.

The primary focus here is **Provider-Assigned (PA) multihoming without NAT**, which preserves the architectural benefits of IPv6. We also touch on translation-based workarounds (NPTv6) and enterprise methods (BGP) for context.

---

## Scenarios and Approaches: What works today?

If you are setting up a multi-WAN IPv6 network, you generally face three architectural choices. Understanding the trade-offs of each is critical before choosing hardware or writing scripts.

### 1. Provider-Assigned (PA) Multihoming (The Standard Way)

In this scenario, your router accepts a delegated prefix (e.g., a `/56` or `/60`) from each ISP and advertises both to your LAN. Your hosts end up with multiple global IPv6 addresses—one for each upstream provider.

- **The Challenge**: If a host sends a packet using ISP A's source address, but the router sends it out ISP B's link, ISP B will likely drop it (ingress filtering/BCP38). Furthermore, hosts often don't know which link is "healthy" and may keep trying to use a source address associated with a down provider.
- **The Solution**: The router must actively manage this complexity. It needs **Source Address Dependent Routing (SADR)** to ensure packets leave the correct interface, and **Conditional Router Advertisements (RAs)** to tell hosts which prefixes are currently valid based on link health.
- **Verdict**: This is the most "correct" IPv6 approach, preserving end-to-end connectivity. However, it requires a router capable of advanced policy routing and dynamic RA management.

### 2. Network Prefix Translation (NPTv6) (The "IPv4-style" Way)

If the complexity of managing multiple source addresses per host is too high, NPTv6 (RFC 6296) offers a middle ground. Similar to IPv4 NAT, the router assigns a stable private (ULA) or public prefix to the LAN. As packets traverse the router, the source prefix is statistically translated 1:1 to match the upstream provider's prefix.

- **The Challenge**: This breaks end-to-end address transparency. Applications that embed IP addresses in their payload, or protocols like IPsec without NAT-T, may break. It also adds state/processing overhead to the router.
- **Verdict**: A reliable stopgap. It avoids the host-side complexity of multiple addresses but sacrifices some IPv6 architectural purity. It is often easier to configure on firewalls like pfSense.

### 3. Provider-Independent (PI) Space + BGP (The Enterprise Way)

This is the "classic" multihoming approach where you own your IP space and announce it via BGP to multiple peers.

- **Verdict**: For most consumer/SMB connections, this is non-viable due to cost, contract requirements, and ISP policies.

---

## Technical Requirements for PA Multihoming

To implement Provider-Assigned multihoming successfully without NAT, your edge router must act as an intelligent mediator between the ISPs and your LAN hosts.

### Routing Outbound Traffic (SADR)

The most critical requirement is **Source Address Dependent Routing**. The router must look at the _source_ IP of every packet leaving the LAN.

- If the source is from ISP A's prefix, route it to ISP A.
- If the source is from ISP B's prefix, route it to ISP B.
- _Note_: This is distinct from standard destination-based routing. Without this, return traffic will be asymmetric or dropped entirely by ISP ingress filters.

### Guiding Hosts (Conditional RAs)

Hosts generally don't check upstream link health; they just use the addresses they are assigned.

- **Preferred Lifetimes**: When an uplink goes down, the router must immediately send a Router Advertisement updating that prefix's Preferred Lifetime to `0`. This tells hosts, "Stop using this address for new connections immediately."
- **Deprecation**: Simultaneously, the router should advertise the surviving ISP's prefix as preferred.
- This mechanism (RFC 8475) allows hosts to fail over to the working connection naturally.

### Health Detection

The router cannot rely on simple interface status (link-up/link-down). It requires active probing (ping/BFD) to multiple targets per ISP. This health state must be tied directly to the routing table (for SADR) and the RA daemon (for host updates).

---

## State of the Ecosystem

Despite the standards being well-defined, "out-of-the-box" support in consumer CPE (Customer Premises Equipment) is sparse.

- **Ubiquiti (UniFi/EdgeRouter)**: As of late 2024, the UniFi ecosystem handles IPv4 failover well but struggles with IPv6. Community reports indicate that failover events often fail to update routing tables or RAs correctly, leaving IPv6 traffic blackholed until a reboot or manual intervention.
- **OpenWrt**: While capable, OpenWrt requires manual configuration. The popular `mwan3` package focuses on connection tracking and policy routing (SADR) but does not natively control `odhcpd` to send conditional Router Advertisements. Users typically rely on custom scripts to bridge this gap.
- **pfSense / OPNsense**: These platforms lean heavily towards NPTv6 (Translation) for multi-WAN IPv6. While stable, it is not a "pure" routing solution.
- **Cisco / Enterprise**: High-end Cisco routers (NCS 500 series, IOS XR) support advanced features like **SRv6 All-Active Multi-Homing** and **EVPN VPWS**, enabling seamless load balancing and redundancy. While overkill for typical home setups, this demonstrates that the technology exists at the carrier/enterprise tier.
- **Commercial/ISP Routers**: Most ISP-provided gateways and consumer mesh systems (Eero, Netgear, etc.) do not support IPv6 multihoming at all. They typically only accept a single upstream delegation.

---

## Resources & References

### Summary of Relevant Standards

| RFC                                                                       | Title                                                              | Relevance to Multihoming                                                        |
| :------------------------------------------------------------------------ | :----------------------------------------------------------------- | :------------------------------------------------------------------------------ |
| **[RFC 3582](https://www.rfc-editor.org/rfc/rfc3582)**                    | Goals for IPv6 Site-Multihoming Architectures                      | Sets the foundational goals: redundancy, load sharing, and scalability.         |
| **[RFC 4218](https://www.rfc-editor.org/rfc/rfc4218)**                    | Threats Relating to IPv6 Multihoming Solutions                     | Discusses security risks like redirection attacks that solutions must mitigate. |
| **[RFC 4219](https://www.rfc-editor.org/rfc/rfc4219)**                    | Things Multihoming in IPv6 Developers Should Think About           | A checklist for developers building multihoming-aware applications or stacks.   |
| **[RFC 5533](https://www.rfc-editor.org/rfc/rfc5533)**                    | Site Multihoming by IPv6 Intermediation (SHIM6)                    | Defines a host-based multihoming protocol (rarely deployed).                    |
| **[RFC 6296](https://www.rfc-editor.org/rfc/rfc6296)**                    | IPv6-to-IPv6 Network Prefix Translation (NPTv6)                    | The standard for stateless prefix translation (the "NAT-like" alternative).     |
| **[RFC 6724](https://www.rfc-editor.org/rfc/rfc6724#section-5)**          | Default Address Selection for IPv6                                 | Defines how hosts choose source IPs (Rule 5.5).                                 |
| **[RFC 7084](https://datatracker.ietf.org/doc/html/rfc7084#section-4.3)** | Basic Requirements for IPv6 Customer Edge Routers                  | Requirement L-13 mandates deprecating prefixes when uplinks fail.               |
| **[RFC 7157](https://www.rfc-editor.org/rfc/rfc7157)**                    | IPv6 Multihoming without Network Address Translation               | Explains the PA multihoming architecture and why NAT should be avoided.         |
| **[RFC 7368](https://datatracker.ietf.org/doc/rfc7368/)**                 | IPv6 Home Networking Architecture Principles                       | Principles for robust home networking, including multi-prefix scenarios.        |
| **[RFC 8028](https://www.rfc-editor.org/rfc/rfc8028)**                    | First-Hop Router Selection in a Multi-Prefix Network               | Updates host behavior to select routers that advertise the source prefix used.  |
| **[RFC 8475](https://datatracker.ietf.org/doc/rfc8475/)**                 | Using Conditional Router Advertisements for Enterprise Multihoming | The definitive guide on using RAs to signal link health to hosts.               |
| **[RFC 8678](https://datatracker.ietf.org/doc/rfc8678/)**                 | Enterprise Multihoming without NPT: Requirements and Solutions     | Detailed requirements for SADR and policy routing in PA multihoming.            |
| **[RFC 8978](https://datatracker.ietf.org/doc/html/rfc8978)**             | Reaction of IPv6 SLAAC to Flash-Renumbering Events                 | Discusses issues when prefixes change rapidly (e.g., failover flapping).        |
| **[RFC 9096](https://datatracker.ietf.org/doc/html/rfc9096)**             | Improving Reaction of Customer Edge Routers to Renumbering         | Updates on how CEs should handle rapid prefix changes.                          |

### Emerging Discussions & Future Directions

- **Multipath Transports (MPTCP / QUIC)**: Recent IETF discussions (e.g., in the IPv6 Working Group) highlight that protocols like **Multipath TCP (MPTCP)** and **QUIC** can inherently handle multihoming at the transport layer by establishing subflows over different paths. This reduces reliance on network-layer failover mechanisms but requires application and server-side support.
- **Multi-Domain IPv6-only Underlays**: New drafts (like "Framework of Multi-domain IPv6-only Underlay Network") explore how operators can run IPv6-only backbones while supporting legacy IPv4 services, which indirectly impacts how multihoming is architected at the edge.
- **SRv6 (Segment Routing over IPv6)**: While primarily a carrier technology, SRv6 is trickling down into enterprise gear (like Cisco IOS XR), offering powerful traffic steering capabilities that could eventually simplify edge multihoming if supported by CPE.

### Community & Platform Threads

- **OpenWrt**: [IPv6 WAN fail-over without IPv6 NAT](https://forum.openwrt.org/t/ipv6-wan-fail-over-without-ipv6-nat/146403/57) – Ongoing discussion on scripting solutions.
- **Ubiquiti**: [Dual WAN IPv6 Failover](https://community.ui.com/questions/Dual-WAN-IPv6-Failover-and-Traffic-Routing-UDM-Pro/8c46d2bb-9aba-422b-ad2d-c78d6a7d5bcb) – User reports on current limitations.
- **pfSense**: [Configuring Multi-WAN for IPv6](https://docs.netgate.com/pfsense/en/latest/recipes/multiwan-ipv6.html) – Official guide utilizing NPTv6.
- **Apple Community**: [Multihomed IPv6 Networks](https://discussions.apple.com/thread/256108158) – Users discussing macOS behavior with multiple prefixes.
