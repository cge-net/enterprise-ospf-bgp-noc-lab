# Design decisions

## Addressing
- **Loopback0 = router ID = management address** (10.255.0.x). The same address identifies a router in OSPF, BGP, SNMP, syslog, traps and NTP. A loopback never goes down while the router runs, and OSPF finds another path to it if one link fails.
- **Documentation ranges** (RFC 5737) for all public addresses and private AS numbers, so nothing collides with real networks.
- **Public /24 split into /26 blocks:** .0/26 DMZ, .64/26 EDGE-1 NAT, .128/26 EDGE-2 NAT, .192/26 spare.

## OSPF
- **Interface mode** (`ip ospf 1 area X`) instead of `network` statements: the area of each interface is visible where the interface is configured – useful on ABRs.
- **Point-to-point network type** on router links: no DR/BDR election on two-router links, faster adjacency.
- **Passive LAN interfaces:** networks are advertised, but hosts cannot form adjacencies.
- **Loopbacks with `ip ospf network point-to-point`** on BR-A/BR-B so their /24 masks are advertised (OSPF otherwise advertises loopbacks as /32), giving summarization real /24s to summarize.
- **Area 1 summarized (`area 1 range 10.1.0.0 255.255.0.0`):** the backbone learns one route for the whole branch, and flapping branch links do not trigger SPF in Area 0. Trade-off: the backbone cannot see a branch subnet fail; traffic is dropped at ABR-A's Null0 route.
- **Area 2 totally stubby (`stub no-summary` on ABR-B, `stub` on BR-B):** single exit, so a default route is as good as a full table. The stub flag must match on both routers (it is carried in hellos); `no-summary` only matters on the ABR, which generates the type-3 LSAs it suppresses.
- **The ISP links are never in OSPF.** OSPF trusts its neighbors completely; the boundary with another organization is BGP's job.

## BGP
- **eBGP on directly connected addresses:** TTL 1 needs no multihop, the ISP has no route to our loopbacks, and the session drops the moment the interface goes down.
- **iBGP on loopbacks with `update-source Loopback0`:** survives the loss of any single internal link. iBGP depends on OSPF to reach the peer's loopback – it cannot provide its own transport.
- **`next-hop-self` on iBGP:** without it, EDGE-1 kept ISP-B's address as next hop and resolved it through its default route via ISP-A.
- **Null0 anchor (`ip route 203.0.113.0 255.255.255.0 Null0`):** a BGP `network` statement only advertises a prefix that exists exactly in the routing table. Internally the more specific /26 still wins; unused addresses are discarded.
- **BGP timers 5/15 toward the ISPs:** the ISP could not see our interface go down and waited the default 180 s hold time. IOS warns that hold times under 20 s increase the risk of flapping – a lab trade-off; production would keep default timers and use BFD.

## Routing policy
- **Local preference 200 on everything from ISP-A** (inbound route-map on EDGE-1). Shared with EDGE-2 over iBGP, so both edges exit via ISP-A. Side effect: local preference is checked before AS-path length, so even ISP-B's own loopback is reached via ISP-A.
- **AS-path prepend ×3 toward ISP-B** (outbound route-map on EDGE-2). Without it, inbound traffic depended on the "oldest path" tie-breaker. Only our own AS is prepended: a foreign AS number would be rejected by that AS (loop prevention) and misrepresents the path.
- **Outbound prefix-list allowing only 203.0.113.0/24** on both edges. Before it, each edge advertised the other ISP's routes – our network offered itself as transit between the providers (a route leak).
- **Direction rule:** to influence outbound traffic, change what you **receive** (local preference inbound); to influence inbound traffic, change what you **advertise** (prepend outbound).

## Default route
- ISPs send a default with `neighbor … default-originate`; each edge redistributes it into OSPF with `default-information originate` **without `always`**. A default is advertised only while it is backed by a working ISP session. With both ISPs down there is no default, so traffic fails at once instead of bouncing between the edges.

## NAT
- **PAT on each edge, different public address per edge.** Both are inside the advertised /24, so BGP needs no change. Different addresses show which edge handled a connection.
- **Inside: links to HQ-CORE and to the other edge** – traffic that reaches EDGE-1 via EDGE-2 (Test 5) is still translated.
- **DMZ excluded** (ACL matches 10.0.0.0/8 only): it already has public addresses.
- **Accepted limitation:** translations exist on one router only. The policy keeps traffic symmetric, so replies return to the router that translated them; during failover and recovery, existing connections reset. A stateful firewall pair would remove this.

## Monitoring
- **Zabbix** (open source, widely used in NOCs) on a separate Ubuntu VM. KVM was not available inside the GNS3 VM, so the server runs beside it in VirtualBox, joined through a host-only network and a GNS3 Cloud node.
- **Dedicated management subnet 10.99.0.0/24**, advertised by OSPF from a passive interface on HQ-CORE. The Zabbix VM routes only 10.0.0.0/8 into the lab; its Internet access (package downloads) uses a separate NAT adapter.
- **Only company routers monitored.** ISP and INET routers belong to other organizations.
- **SNMPv2c read-only + ACL** (permit only 10.99.0.10). The community is a secret global macro – changed in one place.
- **Custom BGP items** instead of relying on the vendor template: one item per session, trigger `<>6` ("not established") rather than `=1` ("idle"), so every abnormal state – including the classic stuck-in-Active – is caught. Severity reflects business impact: losing an ISP session removes redundancy (High); losing iBGP while both ISPs are up only degrades policy (Average).
- **Custom trap template, not changes to the vendor template.** Vendor templates are overwritten on update. The Cisco template already owns `snmptrap.fallback`, so the custom item uses a regex key that catches only link and BGP traps.
- **Traps + syslog + polling together:** push methods are fast but can be lost and are coarse (one item per router); polling is slower but precise (per interface and per peer) and corrects lost messages.
- **NTP from the Zabbix VM (chrony):** a single time source makes logs, traps and graphs comparable.
- **Ubuntu 24.04 LTS rather than 26.04:** Zabbix repository support for 26.04 was still unreliable at build time.
