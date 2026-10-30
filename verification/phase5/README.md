# Phase 5 – iBGP and next-hop-self

| File | Content |
|---|---|
| `p5-before-cef-next-hop-via-isp-a.png` | **Before** `next-hop-self`: CEF on EDGE-1 forwards traffic for ISP-B's link address (198.51.100.1) and ISP-B's loopback (172.31.20.1) out **Ethernet0/0 to ISP-A** – the iBGP next hop was only reachable through the default route, so traffic for a router next to EDGE-2 went out across another provider |
| `p5-edge1-bgp-next-hop-self.png` | EDGE-1 BGP table after `next-hop-self`: every iBGP route now has next hop **10.255.0.2** (EDGE-2's loopback, known through OSPF) instead of ISP-B's 198.51.100.1 |
| `p5-edge1-route-isp-b-loopback-and-8888-paths.png` | `show ip route 172.31.20.1`: learned via iBGP (distance 200) from 10.255.0.2 – ISP-B's router is now reached through EDGE-2 inside our network. `show ip bgp 8.8.8.8`: two paths of equal AS-path length; the **eBGP** one wins ("external, best") |
| `p5-edge1-ibgp-wins-on-as-path-length.png` | `show ip bgp 172.31.20.1`: the **iBGP** path (`65020`, 1 AS) beats the eBGP path (`65010 65030 65020`, 3 ASes) – AS-path length is checked before "eBGP over iBGP". `(metric 11)` is the OSPF cost to EDGE-2's loopback |
| `p5-failover-all-routes-via-edge2.png` | EDGE-1's ISP-A link shut: every route moves to `>i via 10.255.0.2`. The default shows `r>i` (RIB failure) because the OSPF default from EDGE-2 (AD 110) already beats iBGP (AD 200). Earlier in the output, 172.31.20.1 shows only **one** path – ISP-A had picked our path to ISP-B, an early sign of the transit leak fixed in Phase 6 |
