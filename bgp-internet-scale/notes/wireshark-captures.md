# Wireshark Capture Guide — BGP Internet Scale Lab

Three captures back the four answers in the companion video. EVE-NG lets you right-click any link → **Capture** to start a live capture in Wireshark. Filter on `bgp` to isolate BGP messages.

The captures below correspond to specific beats in the companion video. Each one lets you point at a single concrete fact in a real BGP UPDATE message on the wire.

## Important — UPDATEs vs KEEPALIVEs

BGP only sends **UPDATE** messages when a session is establishing or when routes change. A long-running, stable session shows only **KEEPALIVE** messages on the wire (every ~30 seconds). To make UPDATEs flow on demand for the camera, trigger a **soft clear** (route refresh) right before each capture beat. The exact command is listed under each capture below — these are operational-mode commands, not configuration, so they don't violate the hands-off recording rule.

---

## Capture 1 — Inter-AS communication (AS_PATH stamp)

**Link:** PE-A ↔ T1 (`100.64.10.0/30`)
**File:** `cap-1-update-aspath.pcap`

**Trigger UPDATEs on demand** — start the Wireshark capture, then on PE-A:
```
PE-A# clear ip bgp 100.64.10.2 soft out
```
This forces PE-A to re-advertise everything to T1.

**What you'll see:**
- BGP **UPDATE** message from PE-A (`100.64.10.1`) to T1 (`100.64.10.2`) for `203.0.113.0/24`.
- Expand **Path attributes** → **AS_PATH**.
- AS_PATH segment value: `200, 65001`. Two ASes. The "200" is PE-A's AS, stamped on the way out. The "65001" is the originator (CE-1).

**What to point at on camera:** the AS_PATH attribute, specifically the segment containing `200, 65001`. Compare against the AS_PATH on the inbound side (PE-A learned it as `65001` only) — every AS boundary stamps its own number on the way out.

---

## Capture 2 — Path selection (UPDATE attributes)

**Link:** PE-A ↔ CE-1 (`100.64.1.0/30`), inbound to CE-1
**File:** `cap-2-update-attributes.pcap`

**Trigger UPDATEs on demand** — start the Wireshark capture, then on CE-1:
```
CE-1# clear ip bgp 100.64.1.2 soft in
```
This sends a `ROUTE-REFRESH` request to PE-A, which re-sends every UPDATE for this neighbor. Watch them arrive in Wireshark within a second or two.

**What you'll see:**
- BGP **UPDATE** message from PE-A (`100.64.1.2`) to CE-1 (`100.64.1.1`) for `198.51.100.0/24`.
- Expand **Path attributes**:
  - **ORIGIN** — IGP
  - **AS_PATH** — `200, 100`
  - **NEXT_HOP** — `100.64.1.2` (PE-A's interface)
  - **MULTI_EXIT_DISC** (MED) — present or absent depending on PE-A
- **What's NOT in this UPDATE:** `LOCAL_PREF`. LOCAL_PREF is iBGP-only and never crosses an AS boundary on the wire — CE-1 sets it locally via the inbound route-map.

**What to point at on camera:** the path attributes section. Use it to walk through the BGP decision-process tiebreakers — these attributes are exactly what BGP feeds into its 8-step best-path algorithm. Then cut to CE-1's CLI and show the LOCAL_PREF 200 that appeared *after* the route-map ran.

---

## Capture 3 — Filtering (NLRI matches the prefix-list)

**Link:** CE-1 ↔ PE-A (`100.64.1.0/30`), outbound from CE-1
**File:** `cap-3-filtered-update.pcap`

**Trigger UPDATEs on demand** — start the Wireshark capture, then on CE-1:
```
CE-1# clear ip bgp 100.64.1.2 soft out
```
This forces CE-1 to re-send its outbound advertisements to PE-A — the prefix-list-filtered version.

**What you'll see:**
- BGP **UPDATE** message from CE-1 (`100.64.1.1`) to PE-A (`100.64.1.2`).
- Expand **Network Layer Reachability Information (NLRI)** — exactly one prefix: `203.0.113.0/24`.
- Compare against CE-1's full BGP table (`show ip bgp` in CLI) — CE-1 *knows* multiple prefixes (its own /24 plus T1's /24 from both ISPs), but only its own /24 is on the wire.

**What to point at on camera:** the NLRI section. The prefix-list outbound is the single line of policy that prevents CE-1 from accidentally re-advertising T1's prefix back into the global table — i.e., a route leak. One missing line of config is the difference between an enterprise being a customer and accidentally becoming a transit ISP for its provider.

---

## Capture file naming and storage

If you want to ship the captures alongside the lab, save them in a `pcaps/` directory at the root of this repo. Use the filenames above. Anyone replaying the lab can open them in Wireshark without rebuilding the lab.
