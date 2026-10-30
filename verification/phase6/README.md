# Phase 6 – Routing policy (local preference, AS-path prepend, no-transit filter)

## No-transit filter (prefix-list OUR-PREFIX)
| File | Content |
|---|---|
| `p6-edge1-advertised-before-transit-leak.png` | **Before:** EDGE-1 advertises **172.31.20.1/32 (ISP-B's router)** to ISP-A – our AS offered itself as a transit path between the two ISPs |
| `p6-edge1-advertised-after-filter.png` | **After:** only 203.0.113.0/24 is advertised to ISP-A |
| `p6-edge2-advertised-before-transit-leak.png` | **Before:** EDGE-2 advertises **172.31.10.1/32 (ISP-A's router)** to ISP-B |
| `p6-edge2-advertised-after-filter.png` | **After:** only 203.0.113.0/24 is advertised to ISP-B |

## Outbound: local preference 200 on routes from ISP-A
| File | Content |
|---|---|
| `p6-edge1-local-pref-200.png` | Every route from ISP-A carries **LocPrf 200**. Side effect: 172.31.20.1 now goes out via ISP-A (3 ASes) – local preference is checked before AS-path length |
| `p6-edge2-exits-via-edge1.png` | EDGE-2 learned LocPrf 200 over iBGP: best paths for 8.8.8.8 etc. are `>i via 10.255.0.1`, even though it has its own ISP-B paths. The default is `r>i` (RIB failure: the OSPF default from EDGE-1, AD 110, beats iBGP, AD 200) |
| `p6-hq-core-default-only-via-edge1.png` | HQ-CORE's default now comes **only from EDGE-1** (10.0.0.5). EDGE-2 stops originating one because its own default is learned from OSPF |

## Inbound: AS-path prepend toward ISP-B
| File | Content |
|---|---|
| `p6-inet-before-oldest-path-picks-isp-b.png` | **Before:** INET has two equal-length paths (`65010 65001` / `65020 65001`) and picks **ISP-B** – decided by the oldest-path tie-breaker, i.e. by accident |
| `p6-inet-normal-single-path-via-isp-a.png` | **After, normal operation:** INET holds a single path, `65010 65001` via ISP-A – inbound traffic uses the primary ISP by policy, not by accident |
| `p6-dmz-trace-normal-via-isp-a.png` | **After, normal operation:** DMZ-SRV → HQ-CORE → EDGE-1 (10.0.0.5) → ISP-A (192.0.2.1) → INET – outbound also on the primary ISP |
| `p6-inet-failover-prepended-path-only.png` | **ISP-A link down:** INET's only path is the prepended one, `65020 65001 65001 65001 65001`, via ISP-B – the backup is used only when the primary is gone |
| `p6-dmz-trace-failover-via-isp-b.png` | Same failover: DMZ-SRV's traffic leaves via EDGE-2 (10.0.0.9) → ISP-B (198.51.100.1) → INET |

Why INET has only one path in normal operation: ISP-B's own best path to us goes through INET (3 ASes) rather than its direct, prepended link (4 ASes), and a BGP router only advertises its best path.
