# Phase 9 – Failover tests

| File | Content |
|---|---|
| `t4-gns3-edge1-stopped.png` | **Test 4**: EDGE-1 powered off in GNS3 |
| `t4-hq-pc-ping-outage.png` | HQ-PC continuous ping: timeouts while HQ-CORE still forwards to the dead EDGE-1, then a few "unreachable" replies, then traffic via EDGE-2 (~40 s in total) |
| `t4-hq-core-ospf-dead-timer.png` | HQ-CORE drops EDGE-1 only when the **OSPF dead timer** expires – the link stays up in GNS3 |
| `t4-hq-core-default-via-edge2.png` | HQ-CORE's default now originated by EDGE-2 (from 10.255.0.2, via 10.0.0.9) |
| `t4-edge2-bgp-via-isp-b.png` | EDGE-2 uses ISP-B for everything; its iBGP session to EDGE-1 drops only after the 180 s hold time |
| `t4-zabbix-edge1-unavailable.png` | Zabbix: EDGE-1 unavailable by ICMP, EDGE-2 iBGP problems (polled, trap, syslog). No polled ISP-A problem for EDGE-1 – a dead router sends no new data |
| `t4-recovery-edge1-restarted.png` | EDGE-1 back: HQ-CORE's default returns via EDGE-1; Zabbix flags "EDGE-1 has been restarted" |
| `t5-dmz-trace-via-edge2-edge1-isp-a.png` | **Test 5** (HQ-CORE ↔ EDGE-1 link down, both ends shut): DMZ-SRV still exits via the **primary ISP**, one hop longer – HQ-CORE → EDGE-2 (10.0.0.9) → EDGE-1 (10.0.0.1) → ISP-A |
| `t5-hq-pc-trace-and-ping.png` | Same path for HQ-PC; NAT still works because EDGE-1's link to EDGE-2 is a NAT inside interface (TTL 251 – one extra hop) |
| `t5-hq-core-default-via-edge2.png` | HQ-CORE's default is still **originated by EDGE-1** (from 10.255.0.1) but reached via 10.0.0.9; forward metric 10 → 20 |
| `t5-zabbix-no-alert-admin-shutdown.png` | No problem raised: the trap trigger ignores "administratively down" by design, and interface discovery stops monitoring admin-down interfaces (see Phase 8 findings) |
| `t6-both-isp-links-shut.png` | **Test 6**: both ISP links shut, both eBGP sessions drop |
| `t6-hq-core-no-default-route.png` | `% Network not in table` – both edges withdrew the default because neither has one from BGP (`default-information originate` without `always`) |
| `t6-hq-pc-fails-fast-internal-ok.png` | Internet traffic fails **immediately** with "unreachable" from the gateway (no black hole, no loop between the edges); internal traffic to Branch A works |
| `t6-zabbix-both-isp-sessions-down.png` | Zabbix: both ISP sessions **High** (polling), plus trap and syslog problems |

Tests 1–3 (baseline, ISP-link failover, recovery) are shown in the Phase 4, 6, 7 and 8 folders.
