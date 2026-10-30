# Verification screenshots

Each folder has its own README describing every screenshot.

| Folder | Content |
|---|---|
| [phase3](phase3/) | Summarization, Null0 discard route, totally stubby area (16 type-3 LSAs vs 1) |
| [phase4](phase4/) | eBGP sessions, default route into OSPF, first ISP failover |
| [phase5](phase5/) | Next hop before and after `next-hop-self`, iBGP vs eBGP path selection |
| [phase6](phase6/) | Transit leak before/after the filter, local preference, AS-path prepend, inbound path in normal and failover state |
| [phase7](phase7/) | PAT on both edges, DMZ without NAT, failover and recovery with NAT |
| [phase8](phase8/) | Zabbix: management network, SNMP polling, BGP items, syslog, NTP, traps, failover detected three ways |
| [phase9](phase9/) | Failover tests 5 and 6 (internal link loss, both ISPs down) |

Older logs show timestamps starting with `*` (clock not synchronized); NTP was added in Phase 8.
