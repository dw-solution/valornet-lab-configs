# Discontiguous Area 0 — Diagnosis & Fix

## Symptom

Everything *looks* right:
- All interfaces up/up
- OSPF neighbors show **FULL** on every adjacency
- LSDB contains Router LSAs and Type 3 summaries for all remote networks

But:
- **Cisco can ping other Cisco loopbacks**
- **Juniper can ping other Juniper loopbacks**
- **Cisco ↔ Juniper loopback pings fail**
- `show route <remote-loopback>` returns empty on Juniper; `show ip route <remote-loopback>` has no entry on Cisco

## Diagnosis

Look at the Area 0 database on a Juniper router:

```
root@J1> show ospf database

    OSPF database, Area 0.0.0.0
 Type       ID               Adv Rtr           Seq      Age  Opt  Cksum  Len
Router   4.4.4.4          4.4.4.4          0x80000002   696  0x22 0x8878  48
Router   5.5.5.5          5.5.5.5          0x80000003   695  0x22 0xde0d  60
Router  *6.6.6.6          6.6.6.6          0x80000002   747  0x22 0x3dc   48
```

Only `4.4.4.4`, `5.5.5.5`, `6.6.6.6` (the three **Juniper** routers) appear as Router LSAs in Area 0. The Cisco routers `1.1.1.1`, `2.2.2.2`, `3.3.3.3` are missing — even though they're configured as Area 0 on the Cisco side.

## Why

**OSPF Area 0 must be contiguous.** The original topology has:

- Cisco Area 0 island: C1 — C2 — C3
- Juniper Area 0 island: J1 — J2 — J3

The two islands only connect through **Area 1** (C1↔J1) and **Area 2** (C3↔J3). OSPF does not flood Router LSAs across a non-backbone area. Each half of Area 0 sees itself but not the other — a **discontiguous backbone**.

Type 3 Summary LSAs still cross the ABRs into the adjacent non-backbone area, which is why you see `Summary 1.1.1.1` advertised by C1 in Area 1's database. But since those summaries never make it back into the *other* Area 0 island, the routes don't install in the RIB, and pings fail.

## Fix

Two valid options:

### Option 1 — Virtual Link (logical fix)

Tunnel Area 0 across Area 1 between the two ABRs:

**On C1:**
```
router ospf 1
 area 1 virtual-link 6.6.6.6
```

**On J1:**
```
set protocols ospf area 0.0.0.0 virtual-link neighbor-id 1.1.1.1 transit-area 0.0.0.1
commit
```

Virtual links work, but they're an escape hatch — you're admitting the topology violates the rule and papering over it.

### Option 2 — Direct Area 0 cable (physical fix) ⭐ *used in this lab*

Run a backbone cable directly between a Cisco Area 0 router and a Juniper Area 0 router. This stitches the two Area 0 islands into one real, contiguous backbone.

**On C2:**
```
interface Ethernet0/2
 ip address 192.168.1.25 255.255.255.252
 no shutdown

router ospf 1
 network 192.168.1.24 0.0.0.3 area 0
```

**On J2:**
```
set interfaces ge-0/0/1 unit 0 family inet address 192.168.1.26/30
set protocols ospf area 0.0.0.0 interface ge-0/0/1.0
commit
```

## Verification After Fix

```
C2# show ip ospf neighbor
# J2 (5.5.5.5) appears as FULL in Area 0

root@J1> show ospf database
# Now shows Router LSAs for 1.1.1.1, 2.2.2.2, 3.3.3.3 in Area 0.0.0.0

C1# show ip route 4.4.4.4
# O IA route appears

C1# ping 4.4.4.4 source 1.1.1.1
# !!!!!  success
```

## Takeaway

**If OSPF neighbors are FULL but inter-area routes won't install, look at Area 0 first.** LSDB contents on both sides of a suspected break will tell you immediately whether your backbone is actually one area or two.

A clean L3 design puts the backbone where the fastest, most direct paths are — and never lets a non-backbone area become the only transit between Area 0 segments.
