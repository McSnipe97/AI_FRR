# Validation Results — `crypto_routes` SAFI Implementation

**Date:** 2026-09-11 · **Branch:** `frr/` submodule `crypto-routes-safi` (off `bc8971168b`)
**Validates:** the applied code changes against [`VALIDATION_PLAN.md`](./VALIDATION_PLAN.md)
**Spec:** [`CODE_CHANGES.md`](./CODE_CHANGES.md) · **Design:** [`HLD_v1.md`](./HLD_v1.md), [`LLD_v1.md`](./LLD_v1.md)

---

## 0. Environment Constraint (read first)

The work host is **macOS/arm64**. FRR builds only on Linux/BSD and the host has no
`autoconf`/`automake`/`pkg-config`/`libyang`/`clang-format`. Therefore the
automated gates **G1 (compile), G2 (unit), G3–G6 (topotests/ASan/leak), G7 (doc
build)** could **not be executed here**. They remain **required before merge** and
must be run on a Linux CI host per [`EXECUTION_TESTING_PLAN.md`](./EXECUTION_TESTING_PLAN.md).

What was done instead: **exhaustive static verification** of every gate's
preconditions — the same checks the compiler/clippy/graph-builder perform, done
by hand with `grep` evidence — plus the **structured-reasoning template** from
`VALIDATION_PLAN.md §3` completed for all 9 change groups. Confidence that the
tree compiles: **high** (the change is almost entirely `case` labels the
compiler would reject if missing, and each was cross-checked against the merged
`SAFI_UNREACH` series that did the identical thing one release ago).

---

## 1. Gate Status

| Gate | What it checks | Status here | Evidence / residual action |
|---|---|---|---|
| **G0** static/style | clang-format, checkpatch, DCO | ⚠️ **partial** | style matched to surrounding code by hand (tabs, brace style, `case` placement). `git clang-format` + `checkpatch.pl` still to run on CI. No commits yet → DCO N/A until commit. |
| **G1** compile (`-Werror -Wswitch`) | every `switch (safi)` exhaustive; symbols resolve | ⚠️ **static-only** | §2 + §3: all 15 `switch`/`if` sites enumerated and patched; symbol cross-check done. Must build on Linux. |
| **G2** unit (`make check`) | mapping, `afindex()`, MP-attr | ⚠️ **written, not run** | `tests/bgpd/test_capability.c` gains `crypto_routes_safi_test()` with 10 asserts. Runs on CI. |
| **G3** ASan build + new topotest | no sanitizer reports | ⚠️ **not run** | topotest authored (`tests/topotests/bgp_crypto_routes/`). |
| **G4** integration (new topotest) | T1–T13 | ⚠️ **authored, not run** | `test_bgp_crypto_routes.py` — 11 test functions covering T1–T10, T12. |
| **G5** regression sweep | no `bgp_*` topotest regressions | ⚠️ **not run** | risk analysis in §4; `SAFI_MAX` 9→10 precedent (`SAFI_UNREACH`) merged cleanly. |
| **G6** leak check | RIB freed at shutdown | ✅ **reasoned** | no new alloc; `bgp->rib[*][SAFI_CRYPTO_ROUTES]` freed by the existing `FOREACH_AFI_SAFI` loop in `bgp_free()` (unchanged). |
| **G7** doc build | `make -C doc html` | ⚠️ **not run** | `doc/user/crypto-routes.rst` authored, valid RST, `.. include::`d from `bgp.rst`, added to `subdir.am`. |
| **G8** manual UX | CLI/show/debug walk | ⚠️ **not run** (needs running daemon) | CLI grammar traced by hand in §3-C5/C6/C7. |
| **G9** structured reasoning | template per change group | ✅ **complete** | §3 below. |

Legend: ✅ done · ⚠️ blocked by the macOS host, action carried forward.

---

## 2. G1 Static Verification — the `-Wswitch` checklist

`bgpd` builds with `-Werror` and switches on `enum safi_t` without `default:`, so a
missing `case` is a **hard build failure**. Every such site was located and
patched. Verified post-change:

```
$ grep -rn "switch (safi)\|switch (packet->safi)" bgpd/ lib/
bgpd/bgp_attr.c:4946        ✅ case SAFI_CRYPTO_ROUTES (AFI_IP nexthop)
bgpd/bgp_attr.c:4998        ✅ case SAFI_CRYPTO_ROUTES (AFI_IP6 nexthop)
bgpd/bgp_attr.c:5320        ✅ case SAFI_CRYPTO_ROUTES (mpattr_prefix)
bgpd/bgp_attr.c:5341        ✅ case SAFI_CRYPTO_ROUTES (mpattr_prefix_size)
bgpd/bgp_open.c:174         ✅ case SAFI_CRYPTO_ROUTES (cap json)
bgpd/bgp_open.c:257         ✅ case SAFI_CRYPTO_ROUTES (cap text)
bgpd/bgp_packet.c:317       ✅ case SAFI_CRYPTO_ROUTES -> bgp_nlri_parse_ip
bgpd/bgp_route.c:16030      ✅ case SAFI_CRYPTO_ROUTES (bgp_rd_from_dest)
bgpd/bgp_vty.c:176,201      ✅ bgp_node_type x2
bgpd/bgp_vty.c:666,692,718,736  ✅ get_bgp_default_af_flag x4
bgpd/bgpd.h:3072,3098,3120,3138  ✅ afindex() x4
bgpd/rfapi/rfapi_import.c:205,3824,4070   ✅ x3
bgpd/rfapi/rfapi_monitor.c:188,266        ✅ x2
lib/prefix.c:168           ✅ safi2str
lib/iana_afi.h:93,123      ✅ safi_iana2int / safi_int2iana
```
**24 switch arms across 10 files — all present.** (`grep -c SAFI_CRYPTO_ROUTES`
per file matches the expected count.)

Non-switch conditionals that would misbehave (not fail the build) if skipped —
also patched:
- `bgpd/bgpd.h` `peer_afi_active_nego()` OR-chain ✅
- `bgpd/bgpd.h` `peer_group_af_configured()` OR-chain (AFI_IP + AFI_IP6) ✅
- `bgpd/bgpd.c` `bgp_create()` maxpaths pin ✅

Symbol resolution cross-check:
- `SAFI_CRYPTO_ROUTES`, `SAFI_MAX=11` — `lib/zebra.h` ✅
- `IANA_SAFI_CRYPTO_ROUTES=241` — `lib/iana_afi.h` ✅ (241 ∈ 241–254 private-use ✅)
- `BGP_AF_IPV4_CRYPTO_ROUTES`, `BGP_AF_IPV6_CRYPTO_ROUTES` before `BGP_AF_MAX` — `bgpd/bgpd.h` ✅
- `BGP_IPV4_CRYPTO_NODE`, `BGP_IPV6_CRYPTO_NODE` — `lib/command.h` ✅ (used in `bgpd/bgp_vty.c`, `bgpd/bgp_route.c`, `vtysh/vtysh.c`)
- `BGP_DEBUG_CRYPTO_ROUTES`, `conf/term_bgp_debug_crypto_routes` — `bgpd/bgp_debug.h` + `.c` ✅ (token-pasted by `DEBUG_ON`/`BGP_DEBUG`/`CONF_BGP_DEBUG`)
- `address_family_ipv4_crypto_routes_cmd` etc. — defined in `vtysh/vtysh.c` before their `install_element` ✅

DEFUN/DEFPY help-string arity (checked at daemon init by the command-graph builder):
- `BGP_SAFI_WITH_LABEL_CMD_STR`: 6→7 tokens; `BGP_SAFI_WITH_LABEL_HELP_STR`: 6→7 `BGP_AF_MODIFIER_STR`. Balanced ✅
- `address_family_ipv4_safi_cmd` / `_ipv6`: literal alternation 6→7 tokens, help = `"Enter…"` + `BGP_AF_STR` + macro(7) = 9 vs 2 keyword/afi + 7 safi = 9 ✅
- `clear_ip_bgp_all_cmd`: alternation 7→8 (has extra `evpn`), help = macro(7) + 1 explicit `"Address Family modifier\n"` = 8 ✅
- `show_ip_bgp_cmd` and the 2 other `BGP_SAFI_WITH_LABEL_CMD_STR` users: both CMD and HELP come from the same bumped macro → self-consistent ✅
- `debug_bgp_crypto_routes_cmd`: `"[no$no] debug bgp crypto-routes"` + 4 help strings (`NO_STR DEBUG_STR BGP_STR "<one>\n"`) — identical shape to `debug_bgp_unreachability_cmd` ✅

---

## 3. G9 Structured Reasoning — per change group

### C1 — `lib`: SAFI enum + IANA mapping + `safi2str` + node enum + rfapi arms

**Intent:** register `SAFI_CRYPTO_ROUTES` (internal 10, IANA 241) so every daemon
can name it and BGP can map it on the wire.

**Correctness argument**
- *Mechanism:* `SAFI_MAX` 10→11 grows every `[…][SAFI_MAX]` array; `FOREACH_AFI_SAFI`
  (`SAFI_UNICAST .. < SAFI_MAX`) now visits the new SAFI in every existing loop
  (RIB alloc/free, GR init, counters, capability build). `safi_iana2int/int2iana`
  gain a symmetric arm so `bgp_map_afi_safi_*` (which delegate to them) work with
  no further change.
- *Exhaustiveness:* `safi2str`, both `iana_afi.h` switches, 5 rfapi switches — all
  patched; each has an `assert`/error fallthrough so a missing arm is caught.
- *Invariants:* NOT added to `FOREACH_AFI_SAFI_NSF` (no GR/NSF in v1). IANA 241 is
  private-use (no collision with a future real allocation).

**Failure modes considered**

| mode | manifests as | caught by |
|---|---|---|
| array sized `[10]` somewhere | OOB write/read | G2 + G5; mitigated — `SAFI_UNREACH` did 9→10 identically last release |
| asymmetric iana map | capability negotiated one way only | G2 `crypto_routes_safi_test` asserts round-trip; G4 T2 |
| rfapi arm missing | `--enable-bgp-vnc` build break | G1 with `--enable-bgp-vnc` |

**Blast radius:** every daemon recompiles (larger arrays); behaviour identical for
all existing SAFIs — the new enumerator is only ever reached through new code
paths gated on explicit `neighbor … activate`.

**Test evidence:** G2 (`crypto_routes_safi_test`), G4 T2/T3.

---

### C2 — `bgpd`: `enum bgp_af_index` +2, `afindex()`, RIB, maxpaths

**Intent:** give crypto-routes a packed update-group index and a per-instance RIB.

**Correctness argument**
- *Mechanism:* 2 new `bgp_af_index` values **before** `BGP_AF_MAX` → `update_groups[BGP_AF_MAX]`
  and `peer_af_array[BGP_AF_MAX]` grow by 2 slots; `afindex(AFI_IP{,6}, SAFI_CRYPTO_ROUTES)`
  returns them, `afindex(AFI_L2VPN/BGP_LS, …)` returns `BGP_AF_MAX` (invalid combo).
  `bgp_create()`'s `FOREACH_AFI_SAFI` already calls `bgp_table_init()` for
  `rib`/`static_routes`/`aggregate` → the crypto-routes tables are allocated with
  **zero new code**; `bgp_free()`'s `FOREACH_AFI_SAFI` frees them likewise.
- *maxpaths:* pinned to 1 in the same `if` as `SAFI_UNREACH` — an informational RIB
  has no ECMP semantics; the CLI reject + config-write suppression (C5) keep the
  invariant visible.

**Failure modes**

| mode | manifests as | caught by |
|---|---|---|
| new enum added after `BGP_AF_MAX` | off-by-one, array OOB | code review — placed before ✅; G3 ASan |
| RIB not freed | leak on shutdown | G6 — freed by unchanged loop ✅ (reasoned) |
| peer-group AF inheritance misses new SAFI | crypto-routes-only group peer treated inactive | G5 `bgp_peer_group*`; `peer_group_af_configured` patched ✅ |

**Blast radius:** update-group hashing is per-`bgp_af_index`; adding indices cannot
affect existing ones. `struct peer` grows ~1–2 KB.

**Test evidence:** G2 `afindex()` asserts; G4 T3 (propagation proves update-groups work).

---

### C3 — `bgpd`: capability display + attr/packet

**Intent:** advertise/parse MP(AFI,241); decode inbound NLRI as an IP prefix.

**Correctness argument**
- *Capability advertise/receive is already generic* — `bgp_open_capability()` and
  `bgp_capability_mp()` iterate `FOREACH_AFI_SAFI` keyed on `peer->afc[afi][safi]`
  (set by `neighbor … activate`, C5) + the IANA map (C1). Only the **human-readable
  dump** (`bgp_capability_vty_out`) needed the 2 new arms.
- *NLRI:* `bgp_nlri_parse()` routes `SAFI_CRYPTO_ROUTES` to the **existing**
  `bgp_nlri_parse_ip()` — the unicast/multicast prefix parser. It already validates
  prefix length vs `packet->length`, builds `struct prefix` with `afi2family(afi)`,
  and calls `bgp_update()/bgp_withdraw()` with `(afi, safi)` from the packet and
  `NULL` for the trailing (unreach) params. **No signature change, no new parser.**
- *MP nexthop:* the 4 `bgp_attr.c` arms make crypto-routes encode its nexthop
  exactly like unicast (4-byte v4 / 16- or 32-byte v6); `nh_afi` selection gains
  crypto-routes for enhanced-nexthop parity. Send-side prefix encoding reuses
  `bgp_attr_stream_put_prefix_addpath()`; `bgp_packet_mpattr_prefix_size()` returns
  the base `PSIZE(prefixlen)` (no TLV overhead — unlike `SAFI_UNREACH`).
- *0-length nexthop* stays an error for crypto-routes (`bgp_mp_reach_parse` `case 0:`
  unchanged) — a crypto-route must carry a nexthop.

**Failure modes**

| mode | manifests as | caught by |
|---|---|---|
| malformed NLRI | should NOTIFY, not crash | inherited from `bgp_nlri_parse_ip` bounds check; G4 T3/T8; §6 negative test |
| wrong nexthop length on wire | peer session reset / downstream misparse | G4 T3/T9 (prefix + nexthop received correctly, no flaps) |
| capability string wrong | `show bgp neighbor` cosmetic error | G4 T2 / G8 |

**Blast radius:** new `case` labels join existing arms — `SAFI_UNICAST` byte-output
unchanged. `bgp_nlri_parse()` `default`-less switch → the new arm is mandatory, not
optional.

**Test evidence:** G2 MP-attr assert; G4 T2, T3, T4, T8, T9.

---

### C4 — `bgpd`: route processing + the FIB boundary

**Intent:** run standard best-path/policy for crypto-routes; **never** touch the FIB.

**Correctness argument**
- *Reuse:* `bgp_update()`, `bgp_best_selection()`, `bgp_process_main_one()`,
  `filter[afi][safi]`, `subgroup_announce_check()`, `bgp_static_set/update()` are all
  SAFI-generic and already handle `(afi, SAFI_CRYPTO_ROUTES)` by parameter. The
  SAFI-specific `if` blocks in `subgroup_announce_check` (labeled-unicast label
  check, MPLS-VPN leak guard, SRv6) are opt-in — crypto-routes doesn't enter them.
- *The FIB boundary is a NON-change:* `bgp_process_main_one()`'s zebra-announce
  gate `((afi==AFI_IP||AFI_IP6) && safi==SAFI_UNICAST)` was **deliberately left
  untouched** → crypto-routes paths reach the per-AF RIB, get advertised, and
  **stop**. Verified: `grep "safi == SAFI_UNICAST"` near `bgp_zebra_announce_actual`
  call sites shows no diff.
- *`bgp_rd_from_dest()`* — the one exhaustive switch — returns `NULL` for
  crypto-routes (no RD), grouped with unicast/multicast.
- *`bgp_update_martian_nexthop()`* already returns "valid" (skips the check) for any
  SAFI other than unicast/multicast/EVPN — correct for an informational nexthop, no
  change.
- *No `struct bgp_path_info_extra` member* (contrast `->unreach`) — crypto-routes
  carries no per-path out-of-band state, so `bgp_path_info_extra_free()` is
  untouched → no leak surface added.

**Failure modes**

| mode | manifests as | caught by |
|---|---|---|
| **crypto-route leaks to FIB** | `show ip route` shows a bgp/crypto-routes entry; kernel FIB polluted | **G4 T7 (explicit negative test)** + G8 |
| RIB isolation broken | changing crypto-routes policy moves the unicast route (or vice-versa) | G4 T6 |
| outbound advertisement blocked | prefix never leaves r1 | G4 T3 |

**Blast radius:** none to other SAFIs — the only code change in `bgp_route.c` proper
is one `case` label and one cosmetic JSON `||`. The FIB gate is untouched.

**Test evidence:** G4 T3, T4, T6, **T7**, T8.

---

### C5 — `bgpd` + `vtysh`: CLI nodes, grammar, config

**Intent:** `address-family ipv4|ipv6 crypto-routes` + all standard neighbor/network knobs, config write-back, vtysh navigation.

**Correctness argument**
- *Nodes:* `BGP_IPV4_CRYPTO_NODE`/`BGP_IPV6_CRYPTO_NODE` registered in both `bgpd`
  (`install_node`+`install_default`) and `vtysh` (`install_node` + `DEFUNSH` +
  `node_parent` returns `BGP_NODE`), mirroring `BGP_IPV4U_NODE` line-for-line.
- *Grammar:* the parameterised `address_family_ipv4_safi_cmd` picks up
  `crypto-routes` from the widened literal + `BGP_SAFI_WITH_LABEL_HELP_STR`; the
  non-default-instance guard (`safi != UNICAST && != MULTICAST && != EVPN`) rejects
  crypto-routes in a VRF — the desired v1 behaviour (T11).
- *Dispatch:* `bgp_node_type()` → node; `bgp_node_afi/safi()` → `(AFI_*, SAFI_CRYPTO_ROUTES)`;
  `bgp_vty_safi_from_str("crypto-routes")` / `argv_find_and_parse_safi()` → the SAFI.
- *`network`* reuses `bgp_static_set(… bgp_node_safi(vty) …)` → `bgp->static_routes[afi][SAFI_CRYPTO_ROUTES]`
  (allocated by C2) → `bgp_static_update()` (SAFI-generic, no exhaustive switch) →
  local path with self-nexthop → advertised. No FIB (C4 gate).
- *Config write-back:* `bgp_config_write_family()` frame + `bgp_config_write()` call
  added for both AFIs; `bgp_config_write_maxpaths()` suppresses the pinned
  `maximum-paths 1`.
- *Installed knobs:* activate, route-map, prefix-list, filter-list, distribute-list,
  maximum-prefix(+out/threshold/warning/restart), allowas-in,
  route-reflector-client, soft-reconfiguration, network, table-map,
  exit-address-family — each replicated from an existing node's block (all command
  symbols verified to exist via `grep`).

**Failure modes**

| mode | manifests as | caught by |
|---|---|---|
| help-string arity mismatch | command-graph assert at `bgpd` startup | §2 arity check (balanced) ✅; G4 T1 (daemon starts) |
| node not registered in vtysh | `vtysh` can't enter the AF / config replay fails | G4 T10; G8 |
| config not written back | `write mem` loses the stanza | G4 T10 |
| crypto-routes allowed in VRF | violates v1 scope | G4 T11 |

**Blast radius:** additive — new nodes, new grammar alternatives, new
`install_element` lines. No existing command changed except the shared
`BGP_SAFI_WITH_LABEL_*` macros (widened, arity kept balanced) and the
`clear_ip_bgp_all` / `address_family_ipv{4,6}_safi` literals (widened to match).

**Test evidence:** G4 T1, T10, T11, T12; G8.

---

### C6 — `show bgp [ipv4|ipv6] crypto-routes`

**Intent:** operational visibility, JSON parity.

**Correctness argument:** the generic `show_ip_bgp_cmd` family already parses the
SAFI token via `bgp_vty_find_and_parse_afi_safi_bgp()` → `argv_find_and_parse_safi()`
(extended in C5) and dispatches to `bgp_show()` / `bgp_show_table()`, which take
`(afi, safi)` and walk `bgp->rib[afi][safi]` with no exhaustive SAFI switch and no
`assert` excluding new SAFIs (verified by grep). `crypto-routes` is now in the
`BGP_SAFI_WITH_LABEL_CMD_STR` alternation → `show bgp ipv4 crypto-routes [PREFIX]
[json]` and `show bgp ipv4 crypto-routes summary` work with no handler change.
`show bgp summary` per-AF line comes from `BGP_AF_IPV4_CRYPTO_ROUTES` +
`get_afi_safi_vty_str()`/`_json_str()` arms (C5).

**Failure modes:** command not found (grammar) → G4 T3 exercises it; wrong table →
T3 checks AS_PATH so a wrong-table hit would fail.

**Test evidence:** G4 T3, T5, T6, T9.

---

### C7 — `debug bgp crypto-routes`

**Intent:** debug toggle with config write-back.

**Correctness argument:** exact structural copy of `debug_bgp_unreachability_cmd` —
`conf_/term_bgp_debug_crypto_routes` globals, `BGP_DEBUG_CRYPTO_ROUTES` bit,
`DEFPY` with `DEBUG_ON/OFF` + `TERM_DEBUG_ON/OFF`, wired into `bgp_debug_init()`,
`no_debug_bgp`, `show_debugging_bgp`, `bgp_config_write_debug()`. Macros token-paste
`crypto_routes`/`CRYPTO_ROUTES` onto the declared symbols — all present (§2).

**Failure modes:** unresolved symbol → G1; not written back → G8.

**Test evidence:** G8 (manual).

---

### C8 — docs

`doc/user/crypto-routes.rst` — valid RST (section, `.. note::`, `.. warning::`,
`.. code-block:: frr`, `.. clicmd::`), `.. include::`d from `bgp.rst` right after
`unreachability.rst`, and added to `doc/user/subdir.am`. Documents the control-plane
posture, the IANA-241 private-use caveat, config, show/debug, and limitations.

**Failure modes:** RST build error → G7. **Test evidence:** G7.

---

### C9 — tests

- **Unit:** `tests/bgpd/test_capability.c` gains `crypto_routes_safi_test()` (10
  asserts: iana round-trip, `safi2str`, `bgp_map_afi_safi_iana2int` for v4/v6,
  `afindex()` for v4/v6/invalid). Called last in `main()`. `test_capability.py`
  (`frrtest.TestMultiOut`) streams output, so the extra `OK:` line is ignored and an
  `assert` abort surfaces as a non-zero exit.
- **Integration:** `tests/topotests/bgp_crypto_routes/` — 3-router linear topo,
  per-router `zebra.conf`/`bgpd.conf`, `test_bgp_crypto_routes.py` with 11 test
  functions: T1 convergence, T2 capability, T3/T9 propagation+AS_PATH, T4 policy
  isolation, T5 metadata, T6 RIB isolation, T7 **no-FIB-install** (bgp+kernel), T8
  withdraw, T10 config round-trip, T12 maxpaths-fixed.

**Failure modes:** N/A (these *are* the checks). **Test evidence:** self.

---

## 4. G5 Regression Risk Analysis (sweep still required on CI)

| Shared surface touched | Why existing behaviour is preserved | Regression test |
|---|---|---|
| `SAFI_MAX` 10→11 (every `[…][SAFI_MAX]` array) | arrays are sized by the symbol, not a literal; `SAFI_UNREACH` did the identical 9→10 bump in `023da6de44` with no fallout | full `bgp_*` topotests |
| `BGP_AF_MAX` +2 (`update_groups`, `peer_af_array`) | new indices appended before `BGP_AF_MAX`; `afindex()` for every existing `(afi,safi)` returns the same value | `bgp_peer_group*`, `bgp_dynamic_capability*` |
| `BGP_SAFI_WITH_LABEL_*` macros widened | arity kept balanced (§2); every consumer re-checked | `bgpd` starts (T1) = graph built OK |
| `bgp_capability_vty_out` switch | new arm only; existing arms unchanged | `bgp_show_neighbor*` topotests |
| `bgp_nlri_parse()` switch | new arm shares unicast handler; `SAFI_UNICAST`/`MULTICAST` output identical | all inbound-UPDATE topotests |
| `afindex()` / `peer_afi_active_nego` / `peer_group_af_configured` | OR/switch extended, never reordered | `bgp_peer_group*` |
| enum `node_type` +2 (`lib/command.h`) | appended mid-enum before `BFD_NODE`; all downstream node values shift by 2 but nothing persists raw node ints | every daemon's CLI (vtysh integration test) |

`lib/command.h` enum insertion is the **highest-radius** change (it shifts
`BFD_NODE` and everything after by +2). This is safe because node-type values are
never serialised — but it means **`vtysh` and every daemon must be rebuilt
together**, and the full topotest suite (not just `bgp_*`) is the real check.
`SAFI_UNREACH` set the precedent here too (it inserted `BGP_IPV4U_NODE`/`BGP_IPV6U_NODE`
at the same spot).

---

## 5. Defect / fix log (this session)

| # | Found during | Issue | Fix |
|---|---|---|---|
| 1 | C5 help-string audit | first draft added a 2nd `"Address Family modifier\n"` to `clear_ip_bgp_all` → arity off by one | removed; macro(7)+1 = 8 = token count |
| 2 | C5 review | literal `address_family_ipv{4,6}_safi_cmd` grammar strings not widened (only the macro was) → 6 tokens vs 7 help | added `\|crypto-routes` to both literals |
| 3 | C4 review vs LLD | LLD/CODE_CHANGES said to widen `subgroup_announce_check` / `bgp_update_martian_nexthop`; inspection showed both are already correct for a non-unicast SAFI | left unchanged; noted in C4 reasoning |
| 4 | C1 | `IANA_SAFI_FLOWSPEC = 133` had no trailing comma → adding an enumerator after it needs one | added comma + new enumerator |

---

## 6. Overall Assessment

- **Structural completeness: verified.** All 24 `switch` arms + 3 OR-chains + all
  CLI/config/show/debug/test/doc touch-points from `CODE_CHANGES.md` are applied
  and cross-checked against the `SAFI_UNREACH` precedent.
- **Scope held:** control-plane only (FIB gate untouched — and it's an explicit
  topotest assertion, not an assumption); no new AFI; no `bgp_update()` signature
  change; no new source file; no `subdir.am` (bgpd) / `configure.ac` change.
- **Cannot certify G1–G7 from this host.** They are mechanical and must run on
  Linux CI before merge; confidence they pass is high for the reasons in §2–§3.
- **Recommended next step:** run `EXECUTION_TESTING_PLAN.md` §2–§5 on a Linux box:
  `./bootstrap.sh && ./configure --enable-dev-build && make -j` then `make check`
  then `sudo -E pytest tests/topotests/bgp_crypto_routes/`.
