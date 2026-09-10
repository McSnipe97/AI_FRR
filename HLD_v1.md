# HLD v1 — New BGP Address Family `crypto_routes` (FRRouting)

**Status:** Draft v1 · **Date:** 2026-09-10 · **Target:** FRR `master` (`10.8.0-dev`, submodule `bc8971168b`)
**Companion docs:** [`REPO_ANALYSIS_LOG.md`](./REPO_ANALYSIS_LOG.md) (codebase foundation), [`LLD_v1.md`](./LLD_v1.md) (file-level design), [`EXECUTION_TESTING_PLAN.md`](./EXECUTION_TESTING_PLAN.md)
**Problem statement:** [`ProblemStatment.md`](./ProblemStatment.md) — *introduce a new BGP address family (AFI/SAFI) named `crypto_routes`.*

---

## 1. Purpose & Scope

Add a new BGP Subsequent Address Family Identifier, **`crypto_routes`**, to FRR's `bgpd` so that a network of FRR speakers can distribute a dedicated class of IP prefixes ("crypto routes" — prefixes reachable via a crypto/tunnel overlay) in an address family that is **isolated from the unicast RIB** and has its **own import/export policy, its own capability negotiation, and its own operational views**.

### In scope (v1)

- New internal SAFI `SAFI_CRYPTO_ROUTES` under the existing `AFI_IP` and `AFI_IP6`.
- BGP Multiprotocol **capability** advertise/receive for `(IPv4|IPv6, crypto_routes)`.
- **Receive path:** parse `MP_REACH_NLRI` / `MP_UNREACH_NLRI` for the new SAFI, run inbound policy, store in a dedicated per-AF RIB.
- **Best-path** selection in the new RIB (standard algorithm).
- **Advertise path:** run outbound policy, originate `MP_REACH` / `MP_UNREACH` to peers that negotiated the AF.
- **Local origination** via the existing `network` statement.
- **CLI:** `address-family ipv4 crypto-routes` / `address-family ipv6 crypto-routes` config nodes; `neighbor … activate`; `show bgp ipv4|ipv6 crypto-routes …`; `debug bgp crypto-routes`.
- **Config write-back**, `vtysh` integration, user documentation, one unit test, one topotest.

### Out of scope (v1) — see §11

- Installing `crypto_routes` prefixes into the kernel FIB (zebra) — v1 is **control-plane / RIB-only** (same posture as BGP-LS, FlowSpec, `SAFI_UNREACH`).
- A dedicated BGP path attribute for crypto metadata (v1 rides existing extended/large communities).
- VPN (RD-qualified) variant, redistribution from zebra, a brand-new **AFI**, YANG/`mgmtd` northbound modelling, `gRPC`/BMP-specific work beyond what comes for free.

---

## 2. Assumed Semantics of `crypto_routes`

The problem statement does not define the payload, so v1 fixes the **simplest semantics that are useful and let the design reuse the unicast machinery**:

| Property | v1 decision |
|---|---|
| NLRI payload | A plain IPv4 or IPv6 **prefix** — identical wire encoding to unicast NLRI (`<length-octet><prefix-octets>`). |
| Nexthop | Carried in `MP_REACH_NLRI` like unicast; **informational** in v1 (no next-hop tracking, no FIB). |
| Metadata | Carried in existing path attributes (large/extended communities). No new attribute in v1. |
| Consumer | Another controller/daemon that peers with FRR in this AF, or an operator via `show … json` / gRPC. |
| Forwarding | **None.** `crypto_routes` never touches the kernel routing table in v1. |
| Coexistence | A prefix may appear in both `unicast` and `crypto_routes` with no interaction — separate RIBs, separate policy, separate best-path. |

This mirrors how `SAFI_UNREACH` (draft-tantsura, added to FRR April 2026) and BGP-LS behave: a real BGP address family with full protocol plumbing but **no forwarding state**.

---

## 3. Design Principles

1. **Reuse, don't reinvent.** `crypto_routes` NLRI == unicast NLRI, so the receive/advertise/best-path/table code is the *existing* code, reached by adding `case SAFI_CRYPTO_ROUTES:` next to `case SAFI_UNICAST:`. No new parser, no new table type.
2. **New SAFI, not new AFI.** Crypto routes are IPv4/IPv6 prefixes. A new `afi_t` would resize `[AFI_MAX]` arrays across all of `lib/` and *every* daemon and touch `family2afi`/prefix handling — unjustified. (If a non-IP NLRI is ever required, see [LLD Appendix B](./LLD_v1.md#appendix-b--if-a-new-afi-is-truly-required).)
3. **Control-plane only in v1.** Skipping the zebra/FIB path removes the largest and riskiest slice of work and matches the likely intent.
4. **Follow the `SAFI_UNREACH` precedent.** FRR merged a complete "add a SAFI" change in a 15-commit series in 2026 (`023da6de44 … 1b2f34e74f`). The same structure, file set, and commit decomposition apply here — and `crypto_routes` is *simpler* because it reuses unicast NLRI instead of writing a TLV parser.
5. **Compiler-enforced completeness.** `bgpd` builds with `-Wswitch` on `enum safi_t`; every `switch (safi)` without a `default` will fail to compile until the new arm is added. The compiler *is* the change checklist (§ [LLD](./LLD_v1.md)).

---

## 4. Where It Sits in FRR

```
            ┌──────────────────────────── bgpd (one daemon) ────────────────────────────┐
 peer TCP   │                                                                            │
 :179  ───► │  bgp_packet.c ─► bgp_attr.c ─► bgp_nlri_parse()                             │
            │     read task      MP_REACH/     │  switch(safi):                           │
            │                    MP_UNREACH    │   SAFI_UNICAST ┐                          │
            │                                  │   SAFI_CRYPTO_ROUTES ├─► bgp_nlri_parse_ip()   (REUSED)
            │                                  │   SAFI_MULTICAST ┘                        │
            │                                          │                                  │
            │                                  bgp_update() ─► inbound route-map/filters  │
            │                                          │        (per-AF: filter[afi][safi])│
            │                                          ▼                                  │
            │                      bgp->rib[AFI_IP|AFI_IP6][SAFI_CRYPTO_ROUTES]           │
            │                        (dedicated radix trie, auto-allocated in bgp_create) │
            │                                          │                                  │
            │                                  bgp_best_selection()   (REUSED, maxpaths=1)│
            │                                          │                                  │
            │                       ┌──────────────────┴───────────────────┐              │
            │                       ▼                                      ▼              │
            │              bgp_updgrp_*.c  (outbound RM, MP_REACH)   show bgp … crypto-routes / gRPC
            │                       │                                                     │
 peer TCP   │◄──────────────────────┘   MP_REACH_NLRI / MP_UNREACH_NLRI (AFI, SAFI=241)   │
 :179       │                                                                            │
            │   ✗ NO path to zebra / bgp_zebra.c in v1  (control-plane only)              │
            └────────────────────────────────────────────────────────────────────────────┘

 lib/ (libfrr, shared):  afi_t/safi_t enum + IANA mapping + safi2str      ← 3 tiny edits
 zebra/:                 untouched in v1
 vtysh/:                 2 new nodes registered                            ← navigation only
```

---

## 5. (AFI, SAFI) Model

| Layer | Value | Notes |
|---|---|---|
| Internal `safi_t` (`lib/zebra.h`) | `SAFI_CRYPTO_ROUTES = 10`; `SAFI_MAX` 10 → **11** | Dense internal index used to size `[SAFI_MAX]` arrays. |
| Internal `afi_t` | `AFI_IP` (1), `AFI_IP6` (2) — **unchanged** | No new AFI. |
| IANA SAFI (wire) | `IANA_SAFI_CRYPTO_ROUTES = 241` | **Private-Use** range (241–254) per the IANA SAFI registry. v1 deliberately does not squat on an unassigned-but-public number; a production deployment would either request an IANA allocation or make the value configurable (v2). |
| IANA AFI (wire) | 1 / 2 (IPv4 / IPv6) — unchanged | |
| `enum bgp_af_index` (`bgpd/bgpd.h`) | `+ BGP_AF_IPV4_CRYPTO_ROUTES`, `+ BGP_AF_IPV6_CRYPTO_ROUTES` before `BGP_AF_MAX` | Packed index for update-group hashing. |
| CLI token | `crypto-routes` | `safi2str()` → `"crypto-routes"`. |
| `vtysh`/CLI nodes | `BGP_IPV4_CRYPTO_NODE`, `BGP_IPV6_CRYPTO_NODE` (`lib/command.h`) | Mirrors `BGP_IPV4U_NODE` / `BGP_IPV6U_NODE`. |

`SAFI_MAX` moving from 10 to 11 automatically re-sizes ~105 `[AFI_MAX][SAFI_MAX]` arrays in `struct bgp` and `struct peer`, and `bgp_create()`'s `FOREACH_AFI_SAFI` loop automatically allocates `bgp->rib[afi][SAFI_CRYPTO_ROUTES]`. **No per-array code change** — this is why the SAFI axis is the right extension point.

---

## 6. Component View — What Is Touched

| Component | Change class | Size (ref: `SAFI_UNREACH` series) |
|---|---|---|
| `lib/` core AFI/SAFI (`zebra.h`, `iana_afi.h`, `prefix.c`) | Enum + 3 mapping tables + `safi2str` | ~15 lines |
| `bgpd` capability (`bgp_open.c`) | `switch (safi)` display arm; advertise/receive is automatic | ~15 lines |
| `bgpd` attribute/packet (`bgp_attr.c`, `bgp_packet.c`) | `switch (safi)` arms; `bgp_nlri_parse()` dispatch to `bgp_nlri_parse_ip` | ~20 lines |
| `bgpd` route/RIB (`bgp_route.c`, `bgp_route.h`) | `switch (safi)` arms; treat like unicast for policy/best-path; **exclude** from zebra announce gate | ~40 lines |
| `bgpd` core (`bgpd.c`, `bgpd.h`) | `enum bgp_af_index` entries, `afindex()`/`bgp_afi_safi_get_container` arms, maxpaths pin | ~40 lines |
| `bgpd` CLI (`bgp_vty.c`, `bgp_vty.h`) | 2 new nodes, `address-family` grammar, `bgp_node_afi/safi`, `bgp_node_type`, config-write, `show bgp … crypto-routes` | ~200 lines |
| `bgpd` debug (`bgp_debug.c/.h`) | `debug bgp crypto-routes` | ~35 lines |
| `bgpd` misc `switch (safi)` (`bgp_bmp.c`, `bgp_nht.c`, `bgp_fsm.c`, `rfapi/*`) | No-op / "treat as unicast" arms to satisfy `-Wswitch` | ~30 lines |
| `vtysh/vtysh.c` | Register 2 nodes for navigation | ~50 lines |
| `bgpd/subdir.am` | Only if a new `bgp_crypto_routes.c` helper file is added (optional in v1) | ~2 lines |
| `doc/user/` | `bgp.rst` link + `crypto-routes.rst` | new page |
| `tests/` | `tests/bgpd/test_*` assertion + `tests/topotests/bgp_crypto_routes/` | new dir |

**Untouched in v1:** `zebra/`, `staticd/`, `mgmtd/`, `yang/`, all non-BGP daemons, the ZAPI protocol, `lib/zclient.c` (range-checked, not per-value).

---

## 7. Data Flow

### 7.1 Capability negotiation (BGP OPEN)

```
FRR-A                                           FRR-B
  │  OPEN + Cap: MP(AFI=1,SAFI=241)  ───────────►  │  parse via bgp_map_afi_safi_iana2int(1,241)
  │                                                │  → (AFI_IP, SAFI_CRYPTO_ROUTES); afc_recv=1
  │  ◄─── OPEN + Cap: MP(AFI=1,SAFI=241)           │  advertise because afc[AFI_IP][SAFI_CRYPTO_ROUTES]=1
  │  afc_nego[AFI_IP][SAFI_CRYPTO_ROUTES]=1        │  afc_nego=1
  └───────── AF active on the session ────────────►┘
```
Advertise/receive of the MP capability is **already generic** — `bgp_open.c` iterates `FOREACH_AFI_SAFI` and keys off `peer->afc[afi][safi]`, which the `neighbor … activate` CLI sets. Only the *human-readable* capability dump (`show bgp neighbor`) needs a new `switch` arm.

### 7.2 Ingress (peer → local RIB)

1. `bgp_packet.c: bgp_update_receive()` → `bgp_attr_parse()` handles `MP_REACH_NLRI`(14)/`MP_UNREACH_NLRI`(15); `bgp_mp_reach_parse()` reads `(pkt_afi, pkt_safi)`, maps to `(AFI_IP, SAFI_CRYPTO_ROUTES)`.
2. `bgp_nlri_parse()` (`bgp_packet.c:314`) — **new arm** → `bgp_nlri_parse_ip()` (the existing unicast/multicast prefix parser).
3. `bgp_update()` (`bgp_route.c`) runs inbound `filter[AFI_IP][SAFI_CRYPTO_ROUTES]` (route-map / prefix-list / distribute-list / AS-path filter — all already per-`[afi][safi]`).
4. Path installed into `bgp->rib[AFI_IP][SAFI_CRYPTO_ROUTES]`.
5. `bgp_process()` schedules best-path; `bgp_best_selection()` runs unchanged.
6. **v1 stops here** — `bgp_process_main_one()`'s FIB gate `(afi==IP||IP6) && safi==SAFI_UNICAST` is *not* widened, so nothing is sent to zebra.

### 7.3 Egress (local RIB → peer)

1. Best-path change → `group_announce_route()` → per-update-group outbound `filter` + `adv_cmd_rmap[afi][safi]`.
2. `subgroup_announce_check()` / `bgp_updgrp_packet.c` build `MP_REACH_NLRI` with `bgp_map_afi_safi_int2iana(AFI_IP, SAFI_CRYPTO_ROUTES)` → `(1, 241)` and unicast-style prefix + nexthop encoding.
3. Sent only to peers with `peer->afc_nego[afi][SAFI_CRYPTO_ROUTES]`.

### 7.4 Local origination

`address-family ipv4 crypto-routes` → `network 192.0.2.0/24` reuses `bgp_static_set()`, which already takes `bgp_node_safi(vty)`. Wiring = `install_element(BGP_IPV4_CRYPTO_NODE, &bgp_network_cmd)`.

### 7.5 Consumption

`show bgp ipv4 crypto-routes [<prefix>] [detail] [json]` walks `bgp->rib[AFI_IP][SAFI_CRYPTO_ROUTES]` via the existing `bgp_show()` / `bgp_show_table()` with the new SAFI recognised by the `show` command grammar. gRPC/`vtysh -c` come for free.

---

## 8. Non-Functional Requirements

| NFR | How v1 meets it |
|---|---|
| **Correctness** | Reuses proven unicast NLRI/best-path/policy code paths; the delta is `switch` arms the compiler forces you to complete. `maxpaths` pinned to 1 (no multipath ambiguity in an informational RIB). |
| **Scalability** | One extra radix trie per `struct bgp` per AFI (empty until used). `SAFI_MAX` +1 grows fixed `struct peer` arrays by ~1–2 KB/peer — negligible. Update-group machinery already O(peers/group). No new global locks. |
| **Backward compatibility** | New SAFI is opt-in per neighbor (`activate`). Peers that don't support it never see the capability. No change to any existing AF's on-wire behaviour or config. Config file with no `crypto-routes` stanza is byte-identical. |
| **Isolation** | Separate RIB, separate `filter[][]`, separate best-path run, separate update-groups. A misconfigured crypto-routes policy cannot affect unicast. |
| **Security** | Private-use IANA SAFI (no accidental interop with a future real allocation). Control-plane-only → a hostile/incorrect crypto route **cannot install kernel forwarding state**. Standard BGP inbound policy (prefix-list, max-prefix) applies; `neighbor … maximum-prefix` works per-AF unchanged. NLRI length validation is the existing `bgp_nlri_parse_ip()` bounds checking. |
| **Observability** | `show bgp … crypto-routes`, `debug bgp crypto-routes`, `show bgp neighbor` capability line, `show bgp summary` per-AF counters — all from the standard per-AF plumbing. |
| **Operability** | Config write-back + `vtysh` navigation identical to every other AF; no new operational concepts for an operator who knows `address-family`. |

---

## 9. Showcase — Design Delta for the Problem Statement (HLD level)

> This section is the "what changes" showcase requested by the problem statement. LLD-level file/function detail is in [`LLD_v1.md` §3](./LLD_v1.md#3-file-by-file-change-map).

### 9.1 Before → After (capability & families)

```
            BEFORE                                   AFTER (v1)
 address-family ipv4 unicast                 address-family ipv4 unicast
 address-family ipv4 multicast               address-family ipv4 multicast
 address-family ipv4 labeled-unicast         address-family ipv4 labeled-unicast
 address-family ipv4 vpn                     address-family ipv4 vpn
 address-family ipv4 flowspec                address-family ipv4 flowspec
 address-family ipv4 unreachability          address-family ipv4 unreachability
                                        (+)  address-family ipv4 crypto-routes      ◄── NEW
 …ditto ipv6…                                …ditto ipv6… (+) address-family ipv6 crypto-routes

 safi_t: … SAFI_BGP_LS=8, SAFI_UNREACH=9,    safi_t: … SAFI_UNREACH=9,
         SAFI_MAX=10                                 SAFI_CRYPTO_ROUTES=10, SAFI_MAX=11   ◄── NEW
 iana_safi_t: … 133 FLOWSPEC                 iana_safi_t: … 133 FLOWSPEC, 241 CRYPTO_ROUTES ◄── NEW
```

### 9.2 Data-flow delta

| Path | Before | After (v1) |
|---|---|---|
| BGP OPEN capability set | MP caps for negotiated AFs | + optional `MP(1|2, 241)` when neighbor `activate`d in crypto-routes |
| `bgp_nlri_parse()` switch | 8 SAFI arms | + `SAFI_CRYPTO_ROUTES → bgp_nlri_parse_ip()` (shared with unicast) |
| RIBs in `struct bgp` | `rib[AFI][SAFI_MAX=10]` | `rib[AFI][SAFI_MAX=11]` — one more trie per AFI, auto-allocated |
| Inbound/outbound policy | `filter[afi][safi]`, per-AF route-maps | unchanged mechanism; now also reachable for `[afi][SAFI_CRYPTO_ROUTES]` |
| Best-path | per-`(afi,safi)` | + runs for crypto-routes RIB, `maxpaths` pinned to 1 |
| **zebra / kernel FIB** | unicast (+ SRv6 etc.) installed | **unchanged** — crypto-routes deliberately not installed |
| `show` / gRPC / BMP | per-AF | + crypto-routes recognised by `show bgp … crypto-routes` |

### 9.3 Change footprint

```
 ~13 files changed in bgpd/  +  3 files in lib/  +  1 in vtysh/  +  docs  +  tests
 ≈ 450–550 LoC of non-test code (vs. ~4,900 LoC for SAFI_UNREACH, which wrote a TLV parser + UI-RIB)
 0 new dependencies · 0 zebra changes · 0 wire-format invention (reuses unicast NLRI)
```

### 9.4 Reference commit decomposition (mirrors the merged `SAFI_UNREACH` series)

| # | Commit subject | Files |
|---|---|---|
| 1 | `lib: add SAFI_CRYPTO_ROUTES to AFI/SAFI definitions` | `lib/zebra.h`, `lib/iana_afi.h`, `lib/prefix.c`, `bgpd/rfapi/*` |
| 2 | `bgpd: handle crypto-routes SAFI in attribute and packet processing` | `bgp_attr.c`, `bgp_packet.c`, `bgp_route.h` |
| 3 | `bgpd: handle crypto-routes SAFI in capability display` | `bgp_open.c` |
| 4 | `bgpd: add crypto-routes SAFI route processing and update groups` | `bgp_route.c`, `bgpd.h`, `bgpd.c` |
| 5 | `bgpd: add address-family crypto-routes configuration commands` | `bgp_vty.c`, `bgp_vty.h`, `bgpd.c`, `lib/command.h`, `vtysh/vtysh.c` |
| 6 | `bgpd: add 'show bgp [ipv4|ipv6] crypto-routes' command` | `bgp_route.c`, `bgp_vty.c` |
| 7 | `bgpd: add 'debug bgp crypto-routes' CLI` | `bgp_debug.c/.h` |
| 8 | `doc: add BGP crypto-routes SAFI user documentation` | `doc/user/` |
| 9 | `tests: add unit + topotest for crypto-routes SAFI` | `tests/bgpd/`, `tests/topotests/bgp_crypto_routes/` |

---

## 10. Alternatives Considered

| Option | Verdict |
|---|---|
| **New `afi_t` (`AFI_CRYPTO`)** | Rejected. Resizes `[AFI_MAX]` arrays across all of `lib/` + every daemon; forces `family2afi`/`afi2family`/prefix-type work for zero benefit — crypto routes are IP prefixes. |
| **Custom TLV NLRI (like FlowSpec / `SAFI_UNREACH`)** | Rejected for v1. Adds a parser + encoder + a shadow RIB (`bgp_unreach.c` was ~2,500 LoC). Only needed if the payload isn't an IP prefix. Deferred to v2 behind the same SAFI. |
| **New BGP path attribute for crypto metadata** | Deferred. v1 uses large/extended communities. A dedicated attribute needs an IANA type code and interop story. |
| **Install crypto-routes into the kernel FIB** | Deferred to v2. Requires widening `bgp_zebra` gates + a coexistence policy with unicast + a `ZEBRA_ROUTE_*`/table story. Not needed for a distribution-only family. |
| **Model as a VRF / `l3vpn`** | Rejected. That's an operational overlay of existing SAFIs, not a new AF; doesn't match the ask. |
| **Reuse `SAFI_UNREACH`** | Rejected — different semantics (unreachability signalling vs. reachable crypto prefixes) and different IANA identity. |

---

## 11. Risks & Mitigations

| Risk | Likelihood | Mitigation |
|---|---|---|
| Missed `switch (safi)` arm → build break | Medium | `-Wswitch` makes it a *compile* error, not a runtime bug. LLD lists every site; `grep -rn "case SAFI_UNREACH" bgpd/ lib/` enumerates them. |
| `SAFI_MAX` bump breaks an array assumed to be exactly size 10 somewhere | Low | Arrays are all `[SAFI_MAX]`; `SAFI_UNREACH` did the identical 9→10 bump cleanly one release ago. Run full `make check` + topotests. |
| Private-use IANA SAFI 241 collides with an operator's other private use | Low | Documented; v2 makes it configurable. |
| Update-group hashing / `enum bgp_af_index` off-by-one | Low | Add entries *before* `BGP_AF_MAX`; `afindex()` arm covered by unit test. |
| `rfapi` (VNC) switch arms | Low | VNC is `--enable-bgp-vnc` opt-in; add the trivial no-op arms (the `SAFI_UNREACH` commit did exactly this in `rfapi_import.c`/`rfapi_monitor.c`). |
| Reviewer cannot assess correctness before compile | Medium | LLD provides a **pre-compile review checklist** ([LLD §7](./LLD_v1.md#7-pre-compilation-review-checklist)) enumerating each hunk and its expected effect. |

---

## 12. Phasing / Roadmap

| Phase | Content | Exit criteria |
|---|---|---|
| **v1 (this HLD)** | SAFI plumbing, capability, receive/advertise, RIB, policy, CLI, show, one topotest | Two FRR speakers exchange crypto-routes prefixes with independent policy; `make check` + new topotest green; no regression in existing BGP topotests. |
| **v1.1** | `debug`, JSON parity, `show bgp summary` polish, config-replace/`frr-reload` support | `frr-reload.py` round-trips the new stanzas. |
| **v2** | Optional custom NLRI/metadata attribute; configurable IANA SAFI; BMP; graceful-restart NSF participation | Interop test with a second implementation. |
| **v3** | Optional zebra/FIB install path + coexistence policy; YANG model + `mgmtd` | `mgmtd` can configure the AF. |

---

## 13. Acceptance Criteria (v1)

1. `configure && make` succeeds with no new warnings (`-Werror` clean).
2. `router bgp / address-family ipv4 crypto-routes / neighbor X activate` parses, persists across `write memory` + restart, and appears in `show running-config`.
3. Two FRR instances negotiate `MP(1,241)`; `show bgp neighbor` shows the AF as advertised+received.
4. A `network` in crypto-routes on A appears in `show bgp ipv4 crypto-routes` on B, subject to B's inbound route-map, and **does not** appear in `show ip route` (no FIB install) on either.
5. Withdrawing the route on A withdraws it on B (`MP_UNREACH`).
6. Existing `bgp_*` topotests: **no regressions**.
7. New topotest `tests/topotests/bgp_crypto_routes/` passes under `sudo -E pytest`.
8. `git clang-format` and `scripts/checkpatch.pl` clean on the diff; every commit carries `Signed-off-by`.
