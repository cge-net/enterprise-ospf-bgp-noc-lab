# Phase 8 – Monitoring with Zabbix (NOC)

## Management network
| File | Content |
|---|---|
| `p8-topology-mgmt-cloud.png` | GNS3 topology: the MGMT-NOC cloud (10.99.0.0/24) attached to HQ-CORE e1/2, bridged to the Zabbix VM |
| `p8-zabbix-vm-reaches-routers.png` | Zabbix VM: enp0s3 = VirtualBox NAT (downloads), enp0s8 = **10.99.0.10** (lab). Pings to EDGE-1 (TTL 254, 1 hop) and BR-B in the totally stubby area (TTL 253, 2 hops) |
| `p8-mgmt-network-in-ospf.png` | EDGE-1 learns 10.99.0.0/24 via OSPF from HQ-CORE (passive interface – advertised, no hellos to the servers) |

## Zabbix and SNMP polling
| File | Content |
|---|---|
| `p8-dashboard-healthy.png` | Zabbix 7.0 LTS dashboard: server running, 8/8 hosts available (7 SNMP + Zabbix server agent), no problems |
| `p8-hosts-snmp-green.png` | All 7 company routers monitored by SNMP on their Loopback0, templates *Cisco IOS by SNMP* + *Lab SNMP traps*. The ISP and INET routers are not monitored – they belong to other organizations |
| `p8-snmpwalk-edge1.png` | `snmpwalk` of the system group: IOS version, uptime, contact, hostname, location |
| `p8-latest-data-edge1-isp-link.png` | Discovered interface items for EDGE-1 Et0/0. The description **TO_ISP-A** comes from the router's interface description (ifAlias) |

## BGP monitoring (custom items and triggers)
| File | Content |
|---|---|
| `p8-bgp-items-established.png` | Four custom SNMP items (BGP4-MIB `bgpPeerState`, OID 1.3.6.1.2.1.15.3.1.2.<peer>) all **established (6)** via a value map; the syslog log item; the trap item on every router |
| `p8-bgp-triggers.png` | EDGE-1 triggers: ISP-A session **High**, iBGP **Average** (`last(...)<>6`), plus the template's trap trigger with problem and recovery expressions |

## Syslog, NTP, traps
| File | Content |
|---|---|
| `p8-syslog-files.png` | rsyslog on the Zabbix VM: one file per router (named by Loopback0 address) + `all.log`. Branch routers' files were not updated after today's lab start: boot messages sent before OSPF converged were lost (UDP) |
| `p8-ntp-synchronized.png` | HQ-CORE synchronized to the Zabbix VM (chrony), stratum 3 |
| `p8-trap-received.png` | linkDown trap for Ethernet0/0 received by snmptrapd (reason: "administratively down") |

## Final failover: three detection methods compared
EDGE-1's ISP-A link shut at ~13:45:45, restored at ~13:54:25 (PHT). HQ-PC pinging 8.8.8.8 throughout.

| File | Content |
|---|---|
| `p8-failover-trap-syslog-polling.png` | Current problems: **trap 13:45:46 → syslog 13:45:47 → SNMP poll 13:46:01** |
| `p8-failover-graph-edge2.png` | EDGE-2 Et0/0 (TO_ISP-B) bits sent rises from ~300 bps (BGP keepalives) to ~1.1 Kbps when HQ-PC's traffic fails over |
| `p8-failover-resolved.png` | All three problems resolved after `no shut`: trap and syslog at 13:54:27, polling at 13:54:31 |

Push methods (traps, syslog) report within about a second; polling depends on where in its 30-second cycle the failure lands. Polling remains the reliable source: a lost trap or syslog message is corrected by the next poll.

## NOC practice and findings
| File | Content |
|---|---|
| `p8-acknowledged-problem.png` | A problem acknowledged with a note explaining the cause (expected package install), then closed – who handled it and why |
| `p8-lld-disabled-admin-down.png` | After an interface is shut down, Zabbix's interface discovery stops monitoring it ("not discovered anymore … will be deleted in 7 days") |
| `p8-boot-trap-admin-down.png` | False-alarm source found after a reboot: linkDown traps for unused, administratively-down ports (Serial3/3) sent 3 seconds after boot. Fixed by ignoring "administratively down" in the trap trigger |
