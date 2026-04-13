# Multi-Area OSPF — Cisco + Juniper (with Discontiguous Backbone Fix)

Six routers, three OSPF areas, two vendors, one classic design mistake.

This lab builds a multi-area OSPF network across three Cisco and three Juniper routers, then fixes a **discontiguous Area 0** — a textbook violation of OSPF's "backbone must be contiguous" rule that shows up the moment you try to ping across vendors.

🎥 **Companion video:** *(link after upload)*
📺 **Prerequisite video:** Multi-Area OSPF Explained — theory behind the rules you're about to break

---

## Topology

```
                  AREA 1 (192.168.1.20/30)
            +-----------------------------+
            |                             |
        [C1 e0/1]----------------[J1 ge-0/0/1]
          1.1.1.1                  6.6.6.6
            |                             |
     [C1 e0/0]                        [J1 ge-0/0/0]
            |                             |
  192.168.1.0/30                  192.168.1.16/30
       AREA 0                        AREA 0
            |                             |
     [C2 e0/0]                        [J2 ge-0/0/0]
       2.2.2.2                          5.5.5.5
     [C2 e0/2]------ AREA 0 -------[J2 ge-0/0/1]    <-- THE FIX
         192.168.1.24/30  (backbone stitching link)
     [C2 e0/1]                        [J2 ge-0/0/2]
            |                             |
   192.168.1.4/30                 192.168.1.12/30
       AREA 0                         AREA 0
            |                             |
     [C3 e0/1]                        [J3 ge-0/0/1]
       3.3.3.3                          4.4.4.4
     [C3 e0/0]----------------[J3 ge-0/0/0]
            |                             |
            +-----------------------------+
                  AREA 2 (192.168.1.8/30)
```

**Loopbacks (all in Area 0):**

| Router | Loopback0 |
|---|---|
| C1 | 1.1.1.1/32 |
| C2 | 2.2.2.2/32 |
| C3 | 3.3.3.3/32 |
| J1 | 6.6.6.6/32 |
| J2 | 5.5.5.5/32 |
| J3 | 4.4.4.4/32 |

**ABRs:** C1 & J1 (Area 0↔1); C3 & J3 (Area 0↔2)

---

## Subnets

| Link | Subnet | Area |
|---|---|---|
| C1 e0/0 ↔ C2 e0/0 | 192.168.1.0/30 | 0 |
| C2 e0/1 ↔ C3 e0/1 | 192.168.1.4/30 | 0 |
| C3 e0/0 ↔ J3 ge-0/0/0 | 192.168.1.8/30 | **2** |
| J3 ge-0/0/1 ↔ J2 ge-0/0/2 | 192.168.1.12/30 | 0 |
| J2 ge-0/0/0 ↔ J1 ge-0/0/0 | 192.168.1.16/30 | 0 |
| J1 ge-0/0/1 ↔ C1 e0/1 | 192.168.1.20/30 | **1** |
| **C2 e0/2 ↔ J2 ge-0/0/1** | **192.168.1.24/30** | **0** *(stitching link)* |

---

## The Bug (and the Fix)

Without the C2↔J2 link, Area 0 is split into two islands:
- **Cisco Area 0:** C1 — C2 — C3
- **Juniper Area 0:** J1 — J2 — J3

The two halves are only connected through Area 1 (C1↔J1) and Area 2 (C3↔J3). OSPF **refuses** to flood Router LSAs across a non-backbone area, so each half of Area 0 can't see the other. Neighbors go FULL, LSDBs look fine on each vendor side — but inter-vendor pings fail because the routes never install in the RIB.

**Two fixes exist:**
1. **Virtual link** across Area 1 (or Area 2) — stitches the backbone logically
2. **Direct Area 0 cable** between a Cisco and a Juniper — stitches it physically *(this lab uses C2↔J2)*

Option 2 is the cleaner architectural answer. Virtual links are an escape hatch, not a design.

See [`notes/discontiguous-backbone-fix.md`](./notes/discontiguous-backbone-fix.md) for the full walkthrough.

---

## Configs

| Device | File |
|---|---|
| C1 (Cisco, ABR Area 0/1) | [configs/C1.txt](./configs/C1.txt) |
| C2 (Cisco, Area 0 backbone) | [configs/C2.txt](./configs/C2.txt) |
| C3 (Cisco, ABR Area 0/2) | [configs/C3.txt](./configs/C3.txt) |
| J1 (Juniper, ABR Area 0/1) | [configs/J1.txt](./configs/J1.txt) |
| J2 (Juniper, Area 0 backbone) | [configs/J2.txt](./configs/J2.txt) |
| J3 (Juniper, ABR Area 0/2) | [configs/J3.txt](./configs/J3.txt) |

---

## Verification

```
C1# show ip ospf neighbor
C1# show ip route ospf
C1# ping 4.4.4.4 source 1.1.1.1
```

```
root@J1> show ospf neighbor
root@J1> show route protocol ospf
root@J1> ping 1.1.1.1 source 6.6.6.6 count 5
```

Expected: all neighbors **FULL**, all six loopbacks reachable from every router, end-to-end ping succeeds.

---

## Platform

- **EVE-NG Community-CE** on bare metal
- **Cisco IOL / vIOS**
- **Juniper vJunos Routers**
