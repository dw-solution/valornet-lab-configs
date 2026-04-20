# BGP Internet Scale — Cisco + Juniper, 4 routers

A focused 4-router lab that demonstrates the four things BGP does to scale, filter, communicate between autonomous systems, and optimize path selection — with show commands and Wireshark captures, no on-camera configuration.

🎥 **Companion video:** *(link after upload)*

---

## The question this lab answers

> How and why has BGP been so successful and able to scale through the tremendous growth seen on the Internet over the years? What functionality enables BGP to scale, filter, communicate between autonomous systems, and optimize path selection? And how does an enterprise actually set up its connectivity to an ISP?

---

## Topology

```
                       AS 65001 — Enterprise
                       ┌─────────────────────┐
                       │   CE-1 (C8000v)     │
                       │   Lo1: 203.0.113.1  │
                       └──Gi1───────────Gi2──┘
                          │.1            .1│
                100.64.1.0/30        100.64.2.0/30
                          │.2            .2│
                       ┌──Gi1──┐    ┌──ge-0/0/1──┐
                       │       │    │            │
                       │ PE-A  │    │   PE-B     │
                       │C8000v │    │ vJunos     │
                       │AS 200 │    │  AS 300    │
                       │Lo0:   │    │ lo0:       │
                       │10.0.2.1│   │ 10.0.3.1   │
                       │       │    │            │
                       └──Gi2──┘    └──ge-0/0/2──┘
                          │.1            .1│
               100.64.10.0/30        100.64.11.0/30
                          │.2            .2│
                       ┌─Gi0/0──────────Gi0/1┐
                       │   T1 (IOSv)         │
                       │   AS 100            │
                       │   Lo1: 198.51.100.1 │
                       └─────────────────────┘
```

**4 routers, 4 autonomous systems, multi-vendor on the ISP boundary (Cisco PE-A, Juniper PE-B).**

CE-1 is dual-homed into both ISPs. T1 is the Tier-1 transit, originating `198.51.100.0/24` as a stand-in for "the rest of the Internet."

### Devices

| Router  | Vendor          | AS    | Loopback        | Role                         |
|---------|-----------------|-------|-----------------|------------------------------|
| CE-1    | Cisco C8000v    | 65001 | 203.0.113.1/24  | Enterprise edge (dual-homed) |
| PE-A    | Cisco C8000v    | 200   | 10.0.2.1/32     | ISP-A provider edge          |
| PE-B    | Juniper vJunos  | 300   | 10.0.3.1/32     | ISP-B provider edge          |
| T1      | Cisco IOSv      | 100   | 198.51.100.1/24 | Tier-1 transit               |

### Interface map (cable this in EVE-NG)

Each row is one physical link (one cable). Both ends are listed so you can connect them without ambiguity.

| Link | Subnet           | A-side router | A-side iface | A-side IP   | B-side router | B-side iface | B-side IP   |
|------|------------------|---------------|--------------|-------------|---------------|--------------|-------------|
| L1   | 100.64.1.0/30    | CE-1 (C8000v) | **Gi1**      | 100.64.1.1  | PE-A (C8000v) | **Gi1**      | 100.64.1.2  |
| L2   | 100.64.2.0/30    | CE-1 (C8000v) | **Gi2**      | 100.64.2.1  | PE-B (vJunos) | **ge-0/0/1** | 100.64.2.2  |
| L3   | 100.64.10.0/30   | PE-A (C8000v) | **Gi2**      | 100.64.10.1 | T1   (IOSv)   | **Gi0/0**    | 100.64.10.2 |
| L4   | 100.64.11.0/30   | PE-B (vJunos) | **ge-0/0/2** | 100.64.11.1 | T1   (IOSv)   | **Gi0/1**    | 100.64.11.2 |

**Loopbacks (no physical cable):**

| Router | Interface  | IP              | Purpose                                   |
|--------|------------|-----------------|-------------------------------------------|
| CE-1   | Loopback1  | 203.0.113.1/24  | Enterprise prefix (originated into BGP)   |
| PE-A   | Loopback0  | 10.0.2.1/32     | Router-ID                                 |
| PE-B   | lo0.0      | 10.0.3.1/32     | Router-ID                                 |
| T1     | Loopback1  | 198.51.100.1/24 | Tier-1 prefix (originated into BGP)       |

### Interface naming reference

| Vendor / Image       | Physical interface naming convention             |
|----------------------|--------------------------------------------------|
| Cisco C8000v (IOS XE) | `GigabitEthernet1`, `GigabitEthernet2`, …       |
| Juniper vJunos       | `ge-0/0/0`, `ge-0/0/1`, `ge-0/0/2`, …           |
| Cisco IOSv           | `GigabitEthernet0/0`, `GigabitEthernet0/1`, …   |

### Prefixes advertised

| AS    | Prefix          | Originator |
|-------|-----------------|------------|
| 65001 | 203.0.113.0/24  | CE-1       |
| 100   | 198.51.100.0/24 | T1         |

---

## How to use this lab

1. Build the topology in EVE-NG using the interface map above.
2. Paste the entire contents of each file in `configs/` into the corresponding device:
   - `configs/CE-1.txt` → CE-1 (Cisco IOS exec mode)
   - `configs/PE-A.txt` → PE-A (Cisco IOS exec mode)
   - `configs/PE-B.txt` → PE-B (Junos `cli` mode — the script enters `configure exclusive` and ends with `commit and-quit`)
   - `configs/T1.txt`   → T1 (Cisco IOS exec mode)
3. Open Wireshark captures on the three links described in `notes/wireshark-captures.md` (right-click link in EVE-NG → Capture → live Wireshark).
4. Verify with the show-commands tour in the companion video.

The configs install everything the demo needs — no further configuration required to record the video or to follow along.

---

## Verification (after pasting all four configs)

| Router | Command                                  | Expected                                                |
|--------|------------------------------------------|---------------------------------------------------------|
| CE-1   | `show ip bgp summary`                    | 2 neighbors (PE-A, PE-B), both Established              |
| CE-1   | `show ip bgp 198.51.100.0/24`            | 2 paths; best via PE-A with `localpref 200`             |
| CE-1   | `show ip bgp neighbors 100.64.1.2 advertised-routes` | Only `203.0.113.0/24`                         |
| PE-A   | `show ip bgp summary`                    | 2 neighbors (CE-1, T1), both Established                |
| PE-A   | `show ip bgp 203.0.113.0/24`             | AS_PATH `65001`, next-hop `100.64.1.1`                  |
| PE-B   | `show bgp summary`                       | 2 neighbors (CE-1, T1), both Established                |
| PE-B   | `show route 203.0.113.0/24`              | AS_PATH `65001 I`                                       |
| T1     | `show ip bgp summary`                    | 2 neighbors (PE-A, PE-B), both Established              |
| T1     | `show ip bgp 203.0.113.0/24`             | 2 paths; AS_PATH `200 65001` and `300 65001`            |

If any of those don't match, the lab isn't ready — check neighbors first, then the route-maps on CE-1.

---

## What's intentionally NOT in this lab

This lab strips down a richer 7-router design to fit a 15-minute video against a focused discussion question. The omissions, and where they are addressed instead:

- **Route reflectors.** The need is explained on-camera, but no RR is configured because the demo would need 3+ iBGP routers in a single AS to be meaningful. A future video on iBGP scaling will add it.
- **BGP communities.** Skipped to keep the path-selection demo focused on LOCAL_PREF and AS_PATH (which cover 99% of real-world cases). A community demo could be added with one route-map per ISP.
- **Route leaks at scale (the Pakistan-YouTube / Cloudflare incidents).** Mentioned in narration; the prefix-list filter on CE-1 demonstrates the prevention. A separate leak-deep-dive video can re-introduce the on-camera leak demo.
- **AS_PATH prepend.** Inbound LOCAL_PREF on CE-1 covers outbound preference. Prepend (for inbound preference from the rest of the Internet) is a separate teaching point.

If you came looking for any of those, see the notes or wait for the follow-up videos.

---

## Lab files

- `configs/CE-1.txt` — Enterprise edge (Cisco)
- `configs/PE-A.txt` — ISP-A provider edge (Cisco)
- `configs/PE-B.txt` — ISP-B provider edge (Juniper)
- `configs/T1.txt`   — Tier-1 transit (Cisco)
- `notes/wireshark-captures.md` — three Wireshark capture targets keyed to the video beats
