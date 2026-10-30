# Phase 3 – Summarization and totally stubby area

| File | Content |
|---|---|
| `p3-edge1-routes-before-summary.png` | EDGE-1 before summarization: Branch A appears as four separate `O IA` routes (10.1.1.0/24, 10.1.2.0/24, 10.1.3.0/24, 10.1.255.0/30); 22 subnets in 10.0.0.0/8 |
| `p3-edge1-routes-after-summary.png` | EDGE-1 after `area 1 range 10.1.0.0 255.255.0.0` on ABR-A: one `O IA 10.1.0.0/16` (metric 30); 19 subnets. 10.255.0.6/32 (BR-A Lo0) stays separate because it is outside the summary. Area 2 routes stay detailed – totally stubby only filters what enters Area 2 |
| `p3-abr-a-null0-summary-route.png` | ABR-A installs the summary as a discard route to **Null0** (metric 10 = cheapest component) |
| `p3-br-b-routes-totally-stubby.png` | BR-B's whole routing table: its own Area 2 networks plus a single `O*IA 0.0.0.0/0` from ABR-B |
| `p3-br-b-ospf-database-one-lsa3.png` | BR-B's database: two router LSAs and exactly **one** summary LSA (0.0.0.0) |
| `p3-br-a-ospf-database-16-lsa3-normal-area.png` | **Comparison:** BR-A in a normal area (Area 1) holds **16** type-3 summary LSAs from ABR-A – every network outside Area 1 |
| `p3-br-b-stub-area-status.png` | `show ip ospf` on BR-B: "It is a stub area", 1 type-3 LSA |
| `p3-pc-b-to-pc-a-via-default.png` | PC-B reaches PC-A using only the default route (TTL 59 = 64 − 5 routers) |
| `p3-null0-discard-10.1.99.1.png` | HQ-PC to a non-existent address inside the summary: "Destination host unreachable" from **10.0.0.14 (ABR-A)**; the trace stops there – the Null0 route discards it |

**Normal area vs totally stubby area:** BR-A (normal) holds 16 type-3 LSAs; BR-B (totally stubby) holds 1 – a single default route replaces the whole list.
