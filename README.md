# Consumer Edge IPv6 Multihoming (Failover, Load‑Share, and Alternatives)

**Last verified: 2026-09-01.** Platform behavior and IETF drafts move; every
time-sensitive claim below is dated at the point it is made. Corrections welcome.

## Scope & Purpose

This document is for network engineers and advanced users configuring consumer or
SMB edge routers with more than one upstream IPv6 connection.

IPv4 multihoming is trivialized by NAT: the internal network hides behind one
public address per uplink, so failover is a routing change. IPv6 is harder
precisely because it preserves end-to-end addressing — every host holds a
globally routable address derived from a specific provider's prefix. Failover
therefore has to change what addresses hosts *use*, not just where packets go.

The primary focus is **Provider-Assigned (PA) multihoming without NAT**. We also
cover translation-based workarounds (NPTv6 and, separately, NAT66 — they are not
the same thing) and note BGP for context.

### What your uplinks actually give you

Before choosing an architecture, establish what each ISP hands the CE router.
This constrains the design more than any router feature does.

- **A delegated prefix shorter than /64** (DHCPv6-PD, commonly a /56 or /48,
  sometimes a /60). Full design space: a /64 per LAN segment per provider.
- **A single routed /64.** Sufficient for PA multihoming on exactly one LAN
  segment — hosts there hold a global address from each provider and everything
  in this document applies unchanged. It cannot be subdivided, so any additional
  VLAN or downstream router receives nothing from that provider. RFC 7084 L-2
  requires a separate /64 per LAN interface, and the 7084bis draft's LPD-2 and
  LPD-4 expect leftover prefixes to remain available for downstream delegation;
  a single /64 satisfies neither beyond the first segment.
- **A /64 that is on-link rather than routed.** Mobile carriers commonly assign
  the /64 to the WAN link via SLAAC instead of delegating it. Moving it to a LAN
  link then requires RFC 7278-style handoff or ND proxy — an implementation-
  specific capability, not a spec guarantee. UniFi exposes this as "Single
  Network" mode.
- **No usable IPv6 prefix** (IPv4-only service, or a modem that will not pass one
  through). That provider cannot participate in PA multihoming at all;
  translation from a stable internal prefix is the only way that path carries
  IPv6.

Note that prefix size and delivery method are independent constraints. A small
prefix limits how many segments a provider can serve; an on-link assignment
affects whether you can use it on the LAN at all.

---

## The failure mode this document exists to solve

When a dual-stack LAN loses its primary uplink and the router does nothing about
IPv6 addressing, hosts keep a valid-looking GUA from the dead provider's prefix
and a default route to a router that still advertises itself. Two outcomes:

1. **The router keeps forwarding.** Packets exit the surviving WAN with a source
   address from the dead provider's prefix and are dropped by that provider's
   BCP 38 ingress filter. Silent timeouts.
2. **The router drops locally without signaling.** Same silence, one hop earlier.

Either way IPv6 fails *silently* while IPv4 fails over cleanly — so enabling IPv6
makes the outage worse than not having it. Happy Eyeballs v2 masks this for
browsers (they race both families and fall back on connect failure, paying
latency), but Happy Eyeballs is a **per-application** behavior, not a stack
property: glibc's `getaddrinfo` has none, so `apt`, `git`, `docker pull`, mail
clients and most IoT firmware resolve AAAA, connect, and hang. That is the real
blast radius, and it is the argument for fixing this at the gateway.

The correct router behaviors are, in increasing order of effort:

| Behavior | What it fixes | Standard |
| :-- | :-- | :-- |
| RA with Router Lifetime 0 on WAN loss | Removes the router as a default router | RFC 7084 G-5 (unchanged in 7084bis) |
| PIO with **Preferred Lifetime 0** for the stale prefix | Hosts stop *sourcing new connections* from it immediately | RFC 7084 L-13; [RFC 9096 §3.5](https://www.rfc-editor.org/rfc/rfc9096); 7084bis L-13 |
| ICMPv6 Destination Unreachable, **code 5** for packets sourced from an invalidated prefix | Turns silent timeouts into immediate errors for non-Happy-Eyeballs clients | 7084bis L-14 (new) |
| SADR + conditional RAs for a second prefix | Actual dual-provider operation | RFC 8678, RFC 8475 |

---

## Scenarios and Approaches: What works today?

### 1. Provider-Assigned (PA) Multihoming (the standard way)

The router accepts a delegated prefix (a `/56`, `/60`, or `/48`) from each ISP
and advertises both to the LAN. Hosts end up with a global address per provider.

- **The challenge**: if a host sources a packet from ISP A's prefix and the
  router forwards it out ISP B, ISP B drops it at its ingress filter. Hosts have
  no view of uplink health and will keep using an address whose provider is down.
- **The solution**: **Source Address Dependent Routing (SADR)** so packets leave
  by the interface matching their source prefix, plus **conditional Router
  Advertisements** ([RFC 8475](https://datatracker.ietf.org/doc/rfc8475/)) so
  hosts learn which prefixes are currently usable.
- **Verdict**: architecturally correct and preserves end-to-end connectivity.
  Requires a router with per-source policy routing wired to a health check and to
  the RA daemon. No mainstream consumer CPE ships this as a supported feature as
  of September 2026.

### 2. Translation: NPTv6 vs NAT66 (the "IPv4-style" way)

These get conflated constantly, including in vendor UIs. They behave differently
and the difference decides whether inbound connectivity survives.

**NPTv6** ([RFC 6296](https://www.rfc-editor.org/rfc/rfc6296)) is **stateless**,
checksum-neutral, algorithmic 1:1 prefix mapping. No connection tracking. It is
reversible, so inbound works if the mapping is configured.

- Real costs: it still breaks referral-style protocols and address literals in
  payloads; internal and external prefixes must be the same length; inbound
  mappings are pre-configured rather than dynamic; and it breaks IPsec AH.
- It does **not** add per-connection state — that is NAT66's property, not
  NPTv6's.

**NAT66** is stateful, conntrack-based source NAT/PAT — IPv4 masquerading with an
address-family selector. It follows the outbound interface's current address,
which suits failover — the translation follows the active uplink with no
reconfiguration — but there is no deterministic external address for a LAN host,
so every inbound service needs an explicit DNAT rule — per-service port forwarding, on a protocol designed not to need it.

- **Verdict**: NPTv6 is a reasonable stopgap and the only approach where failover
  works without SADR, because the LAN prefix never changes. NAT66 is strictly
  worse for anything you host, and strictly simpler to operate. Check which one
  your platform actually implements — see [State of the Ecosystem](#state-of-the-ecosystem).

### 3. Provider-Independent (PI) Space + BGP (the enterprise way)

Own the address space, announce it to multiple peers. For consumer/SMB links this
is non-viable on cost, contract, and ISP policy grounds. Included only so the
comparison is complete.

---

## Technical Requirements for PA Multihoming

### Routing outbound traffic (SADR)

The router must select the egress interface using the packet's **source** address,
not only its destination:

- Source in ISP A's prefix → route to ISP A.
- Source in ISP B's prefix → route to ISP B.

On Linux this is `ip -6 rule from <prefix> lookup <table>` per uplink — the
mechanism exists everywhere; what is missing in consumer firmware is the
plumbing that keeps those rules in sync with PD state and health checks.
[RFC 8678](https://datatracker.ietf.org/doc/rfc8678/) is the detailed
requirements document.

### Guiding hosts (conditional RAs) — and the valid-vs-preferred trap

This is the single most common implementation error, so be precise when filing a
bug or writing a script:

- **Preferred Lifetime 0 deprecates the address immediately.** RFC 6724 Rule 3
  ("avoid deprecated addresses") makes hosts skip it for *new* connections while
  existing ones drain. There is no clamp on reductions to the preferred lifetime.
- **Valid Lifetime 0 does not do what you expect.** [RFC 4862
  §5.5.3(e)](https://www.rfc-editor.org/rfc/rfc4862) — the "two-hour rule" — says
  a host may not reduce an address's remaining valid lifetime below two hours in
  response to an unauthenticated RA. A naive `valid=0` implementation buys a
  two-hour stall instead of instant invalidation. The rule exists to blunt forged
  deprecation attacks.
- **The correct signal** is a PIO with Preferred Lifetime 0 for the stale prefix.
  RFC 7084 L-13 asks for preferred 0 with valid set to 0 or the lower of the
  current valid lifetime and two hours; RFC 9096 §3.5 asks for both set to 0 and
  re-advertised for at least the previously advertised valid lifetime. Sending
  both zeroed is spec-compliant and safe — just don't *rely* on the valid
  lifetime to be honored quickly by deployed hosts.
- **Cap lifetimes so this matters less.** 7084bis L-15 (MUST) forbids advertising
  LAN lifetimes exceeding the remaining WAN-learned lifetimes, and L-16 (SHOULD)
  points at RFC 9096 §3.4's capped values. You cannot get stuck riding out a
  two-hour lifetime that was never allowed to be advertised.
- **The two-hour rule is being removed.**
  [draft-ietf-6man-slaac-renum](https://datatracker.ietf.org/doc/draft-ietf-6man-slaac-renum/)
  §5.3 formally updates RFC 4862 to let hosts honor small valid lifetimes (-14,
  5 July 2026; WG document, revised I-D needed as of this writing). The Linux
  kernel and NetworkManager already behave this way, per the draft's
  implementation-status section. Don't design around it yet.

### Signaling the failure to clients that ignore RAs

7084bis **L-14** (new relative to RFC 7084) requires the CE router to send ICMPv6
Destination Unreachable **code 5** ("source address failed ingress/egress policy")
for packets forwarded to it using an address from an invalidated prefix. This is
the fix for clients that never see or never act on the RA: they get an immediate
error instead of a timeout. It is also more precisely testable than "send a
deprecation RA": a single packet capture confirms or refutes conformance.

### Health detection

Interface link state is not connectivity. You need active probing (ICMP echo to
multiple off-net targets, or BFD where available) per uplink, and the resulting
state must drive three things: the SADR rules, the RA daemon's per-prefix
lifetimes, and the ICMPv6 error behavior above. A health check that only drives
the IPv4 default route is why IPv6 blackholes on most consumer gear.

---

## Host-Side Reality

Even a perfect router is bounded by host behavior. This is where deployments
actually fail.

- **RFC 6724 Rule 3 is what makes deprecation work.** Rule 5.5 (prefer a source
  address advertised by the next hop you're using) is the piece that makes *two
  simultaneous* prefixes work, and its support is patchy.
  [draft-ietf-6man-rfc6724-update](https://datatracker.ietf.org/doc/draft-ietf-6man-rfc6724-update/)
  (in the RFC Editor queue as of September 2026) introduces a requirement to
  implement Rule 5.5, and also makes "known-local" ULAs preferred over GUAs for
  local traffic — which matters if you deploy ULA + translation.
- **[RFC 8028](https://www.rfc-editor.org/rfc/rfc8028)** tells hosts to select
  the first-hop router that advertised the prefix they're sourcing from. Uneven
  support across Windows, macOS, and Linux; verify rather than assume.
- **Happy Eyeballs is per-application.** Browsers recover; package managers, git,
  mail clients, and IoT firmware do not. Measure with a non-browser client.
- **[RFC 8981](https://www.rfc-editor.org/rfc/rfc8981) temporary addresses** mean
  each host holds several addresses per prefix. Deprecation via PIO covers them
  all (they derive from the same prefix), but per-address hacks do not.
- **Per-host address-family preferences do not generalize.** Windows'
  `DisabledComponents` registry value and the Linux `gai.conf` precedence table
  can force IPv4 preference, but they apply unconditionally rather than during
  outages, are configured per machine, and have no clean macOS equivalent.

## Inbound and DNS

Nothing above helps inbound. If you publish AAAA records for self-hosted
services, failover breaks reachability independently of every mechanism in this
document:

- With PA multihoming, the AAAA is a WAN1 address; on failover it's unreachable
  until DNS is updated and TTLs expire. Short TTLs plus a health-checked DNS
  updater, or don't publish AAAA for services you need during outages.
- With NAT66, there is no deterministic external address at all — per-service
  DNAT rules, per WAN.
- Dynamic DNS driven by the active uplink's prefix is the general answer, and it
  inherits the TTL problem above: reachability returns only as fast as resolvers
  re-query.

---

## State of the Ecosystem

Support in consumer CPE remains sparse. Claims below are dated; re-check before
relying on them.

### Ubiquiti UniFi (UDM / UCG / UXG, UniFi OS)

- **Under the hood** (documented for UDM-class hardware; other UniFi OS models
  share the image but are worth confirming individually): LAN RAs and DHCPv6
  come from **dnsmasq**,
  configured by the UniFi config generator into `/run/dnsmasq.conf.d/` (DHCP bits
  moved to `/run/dnsmasq.dhcp.conf.d/` on newer Network releases). The WAN PD
  client is **odhcp6c**. No radvd, no odhcpd. The UI's RA toggle and RA priority
  map onto dnsmasq's `enable-ra` / `ra-param`.
- **The RA daemon is not the limiting factor.** With
  `dhcp-range=::,constructor:<bridge>,…`, dnsmasq derives advertised prefixes
  from the global addresses actually present on the bridge and follows their
  state: departed prefixes move to an "old prefix" list and are advertised with
  zeroed lifetimes, and its `deprecated` lease-time mode sets the preferred
  lifetime to zero. It will also advertise multiple prefixes on one interface.
  Because it is driven entirely by kernel address state, deprecation follows
  automatically from deprecating or removing the bridge address. The gap is
  above it: UniFi's failover logic operates at the IPv4 route/NAT layer and does
  not touch LAN-side IPv6 addressing, so a stale prefix stays live with a router
  still advertising it.
- **NAT66 is not NPTv6.** UniFi added IPv6 NAT66 rules to the Policy Table in
  Network 9.4.19; per Ubiquiti's own NAT documentation the engine offers SNAT,
  DNAT, and Masquerade, with Masquerade the default — stateful PAT, no stateless
  1:1 prefix mapping. That is worse than pfSense/OPNsense NPTv6 for inbound, and
  fine for outbound-only failover.
- **ULA addressing is available**: the "Additional IPs" option on a VLAN's IPv6
  settings (added in Network 10.0.160, November 2025) allows a segment to carry
  both a PD-derived global prefix and a ULA.
- **No IPv6 failover handling as of Network 10.6.101 (26 August 2026).** IPv6
  work through 2026 has been reporting, WireGuard, DS-Lite, MAP-E, and validation
  fixes; nothing that deprecates prefixes or implements SADR on WAN failover.
- **The config is generated, not authoritative on disk.** The controller
  rewrites the dnsmasq configuration on provisioning, network edits, and firmware
  updates, so local modifications do not survive. Any workaround has to be
  reapplied by an on-boot hook, and can still race a re-provision.

### OpenWrt (25.12 stable; 24.10 old stable as of September 2026)

The most capable option, but you assemble it yourself.

- **odhcpd already implements the standards.** Its RA code implements RFC 9096
  §3.5 explicitly — a prefix that has gone stale is advertised with both Preferred
  and Valid Lifetime zero — and it supports the RFC 9762 P flag and per-interface
  `max_preferred_lifetime` / `max_valid_lifetime` caps. When a PD prefix leaves
  the interface, deprecation happens for free.
- **What's missing is the glue.** `mwan3` is connection-tracking and policy
  routing, mostly IPv4-centric, and does not manage IPv6 prefix state or drive
  odhcpd. SADR is available via `ip -6 rule from <prefix>`; nothing wires it to
  health checks automatically. Expect custom scripts — this is what the OpenWrt
  forum threads below are about.

### pfSense (Netgate)

- Netgate's own multi-WAN IPv6 recipe requires **static IPv6 addressing on all
  WANs** and states plainly that it "does not work for dynamic IPv6 types where
  the subnet is not static, such as DHCP6-PD." Since nearly every consumer ISP
  hands out PD, the documented recipe does not apply to most readers. Gateway
  groups cannot mix address families.
- NPTv6 itself (Firewall → NAT → NPt) is genuine stateless NPTv6, which is better
  than NAT66 — it just needs a stable prefix on both sides.

### OPNsense

- Materially ahead of pfSense here: NPTv6 rules support **Track interface** —
  leave the external prefix empty and it is auto-detected from the tracking
  interface's prefix — so NPTv6 works with dynamic PD. Combined with the newer
  "Identity Association" interface mode for tracking delegated prefixes, this is
  the closest thing to a supported dynamic-PD translation setup on consumer-grade
  software.
- Still translation, not SADR: no conditional-RA logic tied to uplink health.

### ISP-supplied gateways and mesh systems

Eero, Netgear Orbi, carrier CPE and similar generally do not support IPv6
multihoming in any form and accept a single upstream delegation. Assume no.

---

## Resources & References

### Core standards

| RFC | Title | Relevance |
| :-- | :-- | :-- |
| **[RFC 4862](https://www.rfc-editor.org/rfc/rfc4862)** | IPv6 Stateless Address Autoconfiguration | §5.5.3(e) is the two-hour rule — the reason naive `valid=0` deprecation fails. |
| **[RFC 6296](https://www.rfc-editor.org/rfc/rfc6296)** | IPv6-to-IPv6 Network Prefix Translation (NPTv6) | Stateless, checksum-neutral 1:1 prefix mapping. Not NAT66. |
| **[RFC 6724](https://www.rfc-editor.org/rfc/rfc6724)** | Default Address Selection for IPv6 | Rule 3 (avoid deprecated) makes preferred-lifetime-0 work. Rule 5.5 is what makes two live prefixes work, and is the weakly supported part. |
| **[RFC 7084](https://datatracker.ietf.org/doc/html/rfc7084)** | Basic Requirements for IPv6 Customer Edge Routers | G-5 (router lifetime 0 on WAN loss), L-13 (deprecate replaced prefixes), L-2 (a /64 per LAN interface). Being obsoleted — see below. |
| **[RFC 7157](https://www.rfc-editor.org/rfc/rfc7157)** | IPv6 Multihoming without Network Address Translation | The PA multihoming architecture and the case against NAT. |
| **[RFC 7278](https://www.rfc-editor.org/rfc/rfc7278)** | Extending an IPv6 /64 Prefix from a 3GPP Mobile Interface to a LAN Link | How a /64 assigned to a mobile WAN link can be extended to a LAN link. |
| **[RFC 7368](https://datatracker.ietf.org/doc/rfc7368/)** | IPv6 Home Networking Architecture Principles | Multi-prefix home network principles. |
| **[RFC 8028](https://www.rfc-editor.org/rfc/rfc8028)** | First-Hop Router Selection in a Multi-Prefix Network | Hosts should pick the router matching their source prefix. |
| **[RFC 8475](https://datatracker.ietf.org/doc/rfc8475/)** | Using Conditional Router Advertisements for Enterprise Multihoming | The reference design for signaling link health via RAs. |
| **[RFC 8678](https://datatracker.ietf.org/doc/rfc8678/)** | Enterprise Multihoming without NPT: Requirements and Solutions | Detailed SADR and policy-routing requirements. |
| **[RFC 8981](https://www.rfc-editor.org/rfc/rfc8981)** | Temporary Address Extensions for SLAAC | Why each host holds several addresses per prefix. |
| **[RFC 8978](https://datatracker.ietf.org/doc/html/rfc8978)** | Reaction of IPv6 SLAAC to Flash-Renumbering Events | The problem statement for failover flapping and stale prefixes. |
| **[RFC 9096](https://datatracker.ietf.org/doc/html/rfc9096)** | Improving the Reaction of CE Routers to Renumbering Events | §3.4 lifetime caps and §3.5 stale-configuration signaling. BCP 234. |
| **[RFC 9131](https://www.rfc-editor.org/rfc/rfc9131)** | Gratuitous Neighbor Discovery | Reduces packet loss on prefix/router changes; 7084bis L-17. |

### Recent and in-progress work (checked 2026-09-01)

| Document | Status | Why it matters here |
| :-- | :-- | :-- |
| **[draft-ietf-v6ops-rfc7084bis](https://datatracker.ietf.org/doc/draft-ietf-v6ops-rfc7084bis/)** | -06, 6 July 2026. WG document, IESG state "I-D Exists"; the December 2025 milestone to submit to the IESG has slipped. Intended status BCP; obsoletes RFC 7084 **and** RFC 9818. | The single most relevant document. **L-14** (new): ICMPv6 Destination Unreachable code 5 for packets sourced from an invalidated prefix. **L-13** repurposed to "MUST signal stale configuration information as specified in RFC 9096 §3.5". **L-15/L-16**: cap LAN lifetimes to remaining WAN-learned lifetimes. **L-20**: implement SLAAC renumbering per draft-ietf-6man-slaac-renum. **L-21**: RA Guard MUST NOT be on by default. **WPD-3**: accept a delegated prefix smaller than the hint, log an error if it can't address all interfaces. Cite as work in progress. |
| **[RFC 9762](https://www.rfc-editor.org/rfc/rfc9762)** | Standards Track, June 2025 | Defines the PIO **P flag** signaling that the network prefers DHCPv6-PD over per-address assignment; updates RFC 4861/4862. 7084bis LPD-11 says CE routers SHOULD use it. Implemented in odhcpd. |
| **[RFC 9663](https://www.rfc-editor.org/rfc/rfc9663)** | Informational, October 2024 | The per-client DHCPv6-PD deployment model the P flag points at. |
| **[RFC 9818](https://www.rfc-editor.org/rfc/rfc9818)** | Informational, July 2025; updates RFC 7084 | LAN-side prefix delegation on CE routers (LPD-1…LPD-11), folded into 7084bis. Relevant if you have downstream routers competing for the same delegation. |
| **[draft-ietf-6man-rfc6724-update](https://datatracker.ietf.org/doc/draft-ietf-6man-rfc6724-update/)** | -25; approved, RFC Editor queue as of September 2026 | Introduces a **requirement to implement Rule 5.5** and prefers "known-local" ULAs over GUAs for local traffic. Both directly affect multi-prefix source selection. |
| **[draft-ietf-6man-slaac-renum](https://datatracker.ietf.org/doc/draft-ietf-6man-slaac-renum/)** | -14, 5 July 2026; WG document | Successor work to RFC 8978. §5.3 removes RFC 4862's two-hour restriction so hosts can honor small valid lifetimes. Linux kernel and NetworkManager already do. |
| **[draft-ietf-snac-simple](https://datatracker.ietf.org/doc/draft-ietf-snac-simple/)** | -12, 30 August 2026; submitted to the IESG | Stub-network autoconfiguration; the reason 7084bis L-21 forbids RA Guard by default. |

### Not covered here

Multipath transports (MPTCP, QUIC multipath) move failover to the transport layer
and genuinely help, but require server-side support you don't control, so they
are not a substitute for gateway behavior. SRv6 and EVPN multihoming are carrier
technologies for a different problem (PE/CE L2VPN redundancy) and do not address
PA site multihoming at a consumer edge; they are frequently and incorrectly cited
as if they did.

### Community & platform threads

- **OpenWrt**: [IPv6 WAN fail-over without IPv6 NAT](https://forum.openwrt.org/t/ipv6-wan-fail-over-without-ipv6-nat/146403) — scripting approaches.
- **Ubiquiti**: [Dual WAN IPv6 Failover (UDM Pro)](https://community.ui.com/questions/Dual-WAN-IPv6-Failover-and-Traffic-Routing-UDM-Pro/8c46d2bb-9aba-422b-ad2d-c78d6a7d5bcb) — user reports.
- **pfSense**: [Configuring Multi-WAN for IPv6](https://docs.netgate.com/pfsense/en/latest/recipes/multiwan-ipv6.html) — official NPTv6 recipe, static-only.
- **OPNsense**: [NAT documentation, NPTv6 section](https://docs.opnsense.org/manual/nat.html) — "Track interface" for dynamic prefixes.
- **Apple**: [Multihomed IPv6 Networks](https://discussions.apple.com/thread/256108158) — macOS behavior with multiple prefixes.
