# Troubleshooting log

Problems found while building and testing, in the order they occurred. Each follows the same pattern: what was seen, what caused it, how it was fixed, and what it taught.

## Routing

### 1. EDGE-1 had one OSPF neighbor instead of two; loopbacks unreachable
- **Symptom:** EDGE-1 saw only HQ-CORE; ping to ABR-B's loopback failed from EDGE-1 but worked from ABR-B.
- **Cause:** the EDGE-1 ↔ EDGE-2 link and the ABR loopbacks had IP addresses but no `ip ospf 1 area 0`. The one-way ping success was a clue: ABR-B had a route to EDGE-1's loopback, EDGE-1 had none back.
- **Fix:** added the missing interfaces to OSPF.
- **Lesson:** a ping tests both directions – one-way success usually means one side is missing a route.

### 2. iBGP was expected to provide a backup path when OSPF failed
- **Symptom:** with both edges' links to HQ-CORE shut, the edges could not reach each other and no BGP routes appeared.
- **Cause:** the iBGP session runs between loopbacks, which are only reachable through OSPF – and the direct edge-to-edge link was not yet in OSPF (issue 1). When OSPF lost the only path, the BGP session went down too.
- **Lesson:** iBGP rides on top of the IGP; it cannot be its own transport.

### 3. iBGP routes resolved through the other ISP
- **Symptom:** EDGE-1 used routes from EDGE-2 with next hop 198.51.100.1 (ISP-B's address), which is not in OSPF. `show ip route 198.51.100.1` said "not in table", yet the routes were valid.
- **Cause:** iBGP keeps the eBGP next hop. EDGE-1 resolved it through its **default route via ISP-A** – `show ip cef 198.51.100.1` showed next hop 192.0.2.1. Traffic for a router next to EDGE-2 went out across another provider.
- **Fix:** `neighbor … next-hop-self` on both edges, applied with `clear ip bgp … soft out`.
- **Lesson:** `show ip route` shows the table; `show ip cef` shows where packets actually go.

### 4. All neighbors dropped at once on several routers
- **Symptom:** OSPF and BGP neighbors on EDGE-1 died within one second ("dead timer expired", "hold time expired"); a host could not even ARP its gateway on a different router.
- **Cause:** the GNS3 VM stalled (host PC asleep / overloaded), freezing packet forwarding. Not caused by the configuration change made just before.
- **Fix:** restarted the nodes; kept the host awake while testing.
- **Lesson:** when everything fails at the same moment from timers on several devices, check the platform before the last change.

### 5. Inbound traffic used the backup ISP for no visible reason
- **Symptom:** INET chose the path via ISP-B to reach 203.0.113.0/24.
- **Cause:** both paths were two ASes long; BGP fell through to the **oldest eBGP path** tie-breaker (before router ID). Proved by resetting ISP-A's session: the best path moved.
- **Fix:** AS-path prepend toward ISP-B.
- **Lesson:** if the decision reaches the tie-breakers, the design is relying on accidents.

### 6. Failover took about 3.5 minutes
- **Symptom:** after shutting EDGE-1's ISP link, DMZ-SRV could not reach the Internet for minutes. Outbound traffic was fine; replies were lost.
- **Cause:** shutting one end in GNS3 leaves the other end up. ISP-A could not see the failure and kept advertising our prefix until its **180 s BGP hold timer** expired; INET had no other path. Log timestamps: shut 14:10:50, ISP-A noticed 14:13:50.
- **Fix:** `neighbor … timers 5 15` on both edges (the lower hold time is negotiated). Outage dropped to about 10 s (5 lost pings).
- **Lesson:** detection time, not routing, dominates failover. Production: BFD.

### 7. Both ISP sessions flapped right after starting the lab
- **Symptom:** both eBGP sessions dropped and recovered at the same second, while iBGP (default timers) stayed up.
- **Cause:** the GNS3 VM was busy booting all routers; a stall longer than 15 s exceeded the new hold time.
- **Lesson:** this is the risk IOS warns about for hold times under 20 s – a false alarm under load.

### 8. Recovery lost more traffic than the failure (with NAT)
- **Symptom:** continuous ping from HQ-PC: 7 timeouts on failover, 13 on recovery.
- **Cause:** after `no shut`, outbound traffic returned to EDGE-1 immediately (translated to .65), but INET kept sending replies via ISP-B to **EDGE-2, which had no translation for .65**, until the next eBGP update (sent in batches, 30 s advertisement interval). Without NAT (Phase 6) recovery was lossless.
- **Lesson:** with NAT state on separate routers, coming back to the primary can disrupt more than the failure itself.

### 9. A dead edge router took ~40 s to bypass
- **Symptom:** with EDGE-1 powered off, 18 pings timed out, then 6 "unreachable" replies, then traffic flowed via EDGE-2.
- **Cause:** HQ-CORE's link to EDGE-1 stayed up in GNS3, so it waited for the **OSPF dead timer (40 s)**. Then EDGE-2 had to install its own eBGP default before originating one into OSPF.
- **Lesson:** the same detection problem as #6, in a different protocol.

## Monitoring

### 10. Zabbix VM hung at boot
- **Symptom:** Ubuntu stuck at "Loading essential drivers".
- **Cause:** VirtualBox runs on top of Hyper-V on this host (also why KVM was unavailable inside the GNS3 VM); Linux guests with several CPUs can hang in that mode.
- **Fix:** 1 CPU – enough for this workload.

### 11. "Zabbix server is not running"
- **Symptom:** the web interface worked; the server log showed `Access denied for user 'zabbix'@'localhost' (using password: NO)`.
- **Cause:** the `DBPassword` line had been edited but was still commented (`# DBPassword=...`).
- **Fix:** removed the `#`; verified with `grep ^DBPassword`.
- **Lesson:** "using password: NO" pointed straight at the config line, not at the password.

### 12. MySQL stopped
- **Symptom:** web interface error "No such file or directory" (MySQL socket missing).
- **Cause:** MySQL had shut down cleanly ("Server shutdown complete"); no out-of-memory event, no package update at that time. **Cause not confirmed.**
- **Fix:** restarted the service. For next time: check `journalctl --since "-15min"` before restarting.

### 13. Routers never synchronized NTP
- **Symptom:** routers showed reach 0, stratum 16, while `timedatectl` on the VM said "synchronized: yes".
- **Cause:** Ubuntu's built-in time service (systemd-timesyncd) is a client only – it keeps the VM's clock correct but does not serve time.
- **Fix:** installed chrony with `allow 10.0.0.0/8` (it replaced timesyncd).
- **Lesson:** "this machine has the right time" and "this machine provides time" are different roles.

### 14. Traps not written to the trap file
- **Symptom:** snmptrapd log: `Cannot open [/var/log/snmptrap/snmptrap.log]: Permission denied`.
- **Cause:** snmptrapd runs as user `Debian-snmp` (`User=` in its systemd unit), not root.
- **Fix:** `chown Debian-snmp:Debian-snmp /var/log/snmptrap`.

### 15. Trap template could not be linked
- **Symptom:** "item with key snmptrap.fallback already exists" when linking the custom template.
- **Cause:** the Cisco template already had a fallback trap item, so traps had been arriving (but with no triggers).
- **Fix:** custom item uses a regex key `snmptrap["6\.3\.1\.1\.5\.|2\.1\.15\.7\."]` for link and BGP traps; everything else still reaches the Cisco fallback item.
- **Also:** the BGP trap OIDs were `1.3.6.1.2.1.15.7.1/.2` on this IOS, not the `15.0.x` values first assumed – confirmed by reading real traps before writing triggers.

### 16. False "link down" problems after every lab start
- **Symptom:** five routers showed "link down (trap)" that never cleared, although all links were up.
- **Cause:** at boot, IOS sends linkDown traps for every unused, administratively-down port (e.g. Serial3/3, 3 seconds after boot). The trigger read only the latest trap of one item shared by all interfaces.
- **Fix:** trigger ignores traps containing "administratively down"; manual close allowed on trap triggers (a recovery trap may never arrive).
- **Lesson:** test monitoring through a full restart.

## Other observations
- **Package-change warnings** on the Zabbix server matched the apt history exactly (installs at 18:33 and 18:51; the second only detected at the first hourly check after the next boot). Acknowledged with a note and closed – the expected NOC workflow.
- **Interface discovery removes admin-down interfaces:** a planned shutdown of an in-use link disables its items without any alert, and deletes them after 7 days. Planned work should use a maintenance period.
- **Syslog boot messages lost:** routers send logs before OSPF has a route to the server; UDP drops them silently.
- **A dead router raises no polled BGP problem:** with no new data, `last()` keeps the old "established" value. The ICMP / availability checks catch it; `nodata()` would close the gap.
