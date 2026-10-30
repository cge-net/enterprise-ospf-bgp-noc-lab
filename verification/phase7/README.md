# Phase 7 – NAT (PAT) on both edge routers

Each edge translates internal 10.0.0.0/8 traffic to its own public address: EDGE-1 → **203.0.113.65**, EDGE-2 → **203.0.113.129**. The DMZ (203.0.113.0/26) already uses public addresses and is not translated.

| File | Content |
|---|---|
| `p7-hq-pc-reaches-internet.png` | HQ-PC (10.0.10.10) now reaches 8.8.8.8 – it failed in Phase 4 |
| `p7-edge1-pat-translations.png` | EDGE-1 `show ip nat translations`: HQ-PC (10.0.10.10) and PC-A (10.1.1.10) both appear as **203.0.113.65**, told apart by port number (PAT / overload) |
| `p7-pc-a-pc-b-reach-internet.png` | Both branch PCs reach 8.8.8.8. PC-B uses only its totally-stubby default route to get there |
| `p7-dmz-not-translated.png` | DMZ-SRV reaches 8.8.8.8 while **neither** edge shows a translation – the NAT-INSIDE ACL only matches 10.0.0.0/8 |
| `p7-edge1-nat-statistics.png` | Outside: Et0/0; inside: Et0/1 and Et0/2 (the inter-edge link is "inside" so traffic arriving via EDGE-2 is still translated); 30 hits, 0 misses |
| `p7-failover-and-recovery-ping.png` | HQ-PC continuous ping. **Failover** (ISP-A link shut): 7 timeouts (~14 s). **Recovery** (`no shut`): 13 timeouts (~26 s) – *longer* than the failure (see below) |
| `p7-edge2-translations-after-failover.png` | After failover EDGE-2 translates HQ-PC to **203.0.113.129** – the public address shows which edge handled the traffic |
| `p7-inet-cannot-reach-nat-address.png` | INET → 203.0.113.65 fails (`U`): no translation exists for a connection started from outside, so EDGE-1 drops it at the Null0 /24 route. PAT only accepts replies to connections started inside |

**Why recovery lost more pings than the failure:** after `no shut`, outbound traffic moves back to EDGE-1 at once (local preference 200) and is translated to .65. Inbound needs EDGE-1's /24 to reach INET via ISP-A, and IOS sends eBGP updates in batches (30-second advertisement interval). Until then INET still returns traffic via ISP-B to **EDGE-2, which has no translation for .65** and drops it. NAT state that lives on one router makes recovery asymmetric for a short time – a firewall pair with state synchronization avoids this in production.
