# Phase 4 – eBGP to the ISPs

| File | Content |
|---|---|
| `p4-edge1-bgp-summary.png` | EDGE-1 `show ip bgp summary`: iBGP to EDGE-2 (10.255.0.2) and eBGP to ISP-A (192.0.2.1) both up, 5 prefixes received from each |
| `p4-edge1-bgp-table.png` | EDGE-1 BGP table: default, 8.8.8.8/32 via `65010 65030`, and our own 203.0.113.0/24 (next hop 0.0.0.0, weight 32768). The iBGP copies still carry next hop **198.51.100.1** (ISP-B's address) – the problem fixed in Phase 5 with `next-hop-self` |
| `p4-inet-two-paths-to-our-prefix.png` | INET has **two** paths to 203.0.113.0/24, both two ASes long (`65010 65001` and `65020 65001`). The best one was chosen by the **oldest-path** tie-breaker – inbound traffic depended on an accident, fixed by AS-path prepending in Phase 6 |
| `p4-hq-core-default-via-both-edges.png` | HQ-CORE: `O*E2 0.0.0.0/0` from **both** edges (10.0.0.5 and 10.0.0.9) – each edge originates the default into OSPF while it has one from its ISP |
| `p4-br-a-external-default.png` | BR-A also gets `O*E2 0.0.0.0/0`: Area 1 is a normal area and accepts external (type 5) routes |
| `p4-dmz-srv-reaches-internet.png` | DMZ-SRV (public 203.0.113.10) reaches 8.8.8.8 |
| `p4-hq-pc-fails-no-nat-yet.png` | HQ-PC (private 10.0.10.10) fails: the request leaves, but INET has no route back to 10.x. Fixed by NAT in Phase 7 |
| `p4-failover-isp-a-link-shut.png` | EDGE-1's ISP-A link shut: BGP session drops immediately ("Interface flap"), DMZ-SRV still reaches 8.8.8.8 via EDGE-2 / ISP-B |

Timestamps in these logs start with `*` (clock not synchronized) – NTP was added in Phase 8.
