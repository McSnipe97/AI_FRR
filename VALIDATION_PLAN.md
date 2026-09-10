# Validation Plan — `crypto_routes` SAFI Code Changes

**Status:** v1 · **Date:** 2026-09-10
**Validates:** [`CODE_CHANGES.md`](./CODE_CHANGES.md) · **Runbook:** [`EXECUTION_TESTING_PLAN.md`](./EXECUTION_TESTING_PLAN.md) · **Context:** [`HLD_v1.md`](./HLD_v1.md), [`LLD_v1.md`](./LLD_v1.md)

Where `EXECUTION_TESTING_PLAN.md` is the *sequential runbook* (env → build → test), this document is the *acceptance argument*: for every code change, what could go wrong, how it is proven correct, and the pass/fail gate. It also defines the **structured-reasoning template** to be applied when the changes are made.

---

## 1. Validation Principles

1. **The compiler is the first validator.** `-Wswitch -Werror` on `enum safi_t` means an omitted `case` is a build failure, not a latent bug. Every switch site in [`CODE_CHANGES.md` §11](./CODE_CHANGES.md#11-exhaustive-switchif-site-checklist-verified-against-bc8971168b) must be accounted for before the tree links.
2. **Reuse implies inheritance of correctness.** crypto-routes routes through `bgp_nlri_parse_ip()`, `bgp_best_selection()`, `bgp_attr_stream_put_prefix_addpath()`, `filter[afi][safi]` — code already covered by the existing unicast test suite. Validation focuses on the *seams* (dispatch arms, enum sizing, CLI wiring, the FIB boundary), not on re-testing unicast.
3. **Prove the negative.** The defining property of v1 is *"never touches the FIB"*. That is an explicit test assertion (T7), not an assumption.
4. **No regression is a first-class requirement.** `SAFI_MAX` 10→11 and `BGP_AF_MAX` +2 touch shared arrays; the full `bgp_*` topotest set must stay green.
5. **Every change traces to a test.** §4 is the traceability matrix; no change ships without a row.

---

## 2. Validation Layers & Gates

| Gate | Layer | Tooling | Blocks merge? | Runtime |
|---|---|---|---|---|
| G0 | Static / style | `git clang-format`, `scripts/checkpatch.pl`, DCO check | yes | seconds |
| G1 | Compile | `configure && make` with gcc **and** clang, `-Werror` | yes | ~15–25 min |
| G2 | Unit | `make check` (incl. new asserts) | yes | ~2 min |
| G3 | Sanitizer build | `--enable-address-sanitizer` build + targeted topotest | yes | +topo |
| G4 | Integration (new) | `tests/topotests/bgp_crypto_routes/` | yes | ~60–90 s |
| G5 | Regression | full `tests/topotests/bgp_*` + `evpn_*`, `bgp_gr_*` | yes | ~1–3 h |
| G6 | Leak | `--valgrind-memleaks` on the new suite | yes | ~10 min |
| G7 | Doc build | `--enable-doc && make -C doc` | yes | ~2 min |
| G8 | Manual UX | `--pause` topo shell, CLI/`show`/`debug` walk | reviewer sign-off | manual |
| G9 | Structured reasoning review | §6 template completed per change group | reviewer sign-off | manual |

---

## 3. Structured-Reasoning Template (apply per change group when implementing)

For each of the 9 commit groups in `CODE_CHANGES.md`, the implementer records:

```
### Change group: <Cn — title>

**Intent:** <one sentence — what this group makes work>

**Files/hunks:** <list>

**Correctness argument:**
- Precondition: <what must already be true — e.g. "SAFI_CRYPTO_ROUTES defined, iana mapping present">
- Mechanism: <why the change produces the intended behaviour — cite the reused function>
- Exhaustiveness: <every switch/if touched; grep evidence>
- Invariants preserved: <SAFI_MAX sizing, no FIB path, capability opt-in, no attr change>

**Failure modes considered:**
| Failure mode | Would manifest as | Caught by |
|---|---|---|
| ... | ... | G1/G2/G4/... |

**Blast radius:** <which other SAFIs/AFs/daemons this could affect, and why it doesn't>

**Test evidence:** <gate + specific assertion IDs that exercise this group>
```

A completed example for **C3.3 (`bgp_nlri_parse()` dispatch)**:

> **Intent:** inbound crypto-routes NLRI in an UPDATE is decoded as an IP prefix.
> **Mechanism:** `bgp_nlri_parse()` gains `case SAFI_CRYPTO_ROUTES:` sharing the `SAFI_UNICAST` arm → `bgp_nlri_parse_ip(peer, attr, packet)`, which already validates prefix length against `packet->length`, builds `struct prefix` with `afi2family(afi)`, and calls `bgp_update()/bgp_withdraw()` with `(afi, safi)` from the packet and `NULL` for the trailing (unreach) params. No new parser, no signature change.
> **Exhaustiveness:** `bgp_nlri_parse()` is the only `switch (packet->safi)` (grep). It has no `default:` → the arm is mandatory for the build.
> **Invariants:** prefix decode identical to unicast; the routes land in `bgp->rib[afi][SAFI_CRYPTO_ROUTES]` (separate trie); nothing calls `bgp_zebra_*`.
> **Failure modes:** malformed NLRI → `bgp_nlri_parse_ip` returns `BGP_NLRI_PARSE_ERROR` → NOTIFICATION (inherited). Oversized prefixlen → existing `p.prefixlen > prefix_blen` check. Both exercised by `test_mp_attr` + topotest T4/T8.
> **Blast radius:** none — new `case` label only; `SAFI_UNICAST`/`SAFI_MULTICAST` behaviour byte-identical.

---

## 4. Change → Validation Traceability Matrix

| Change (CODE_CHANGES ref) | Risk if wrong | Primary validation | Pass criterion |
|---|---|---|---|
| C1.1 `SAFI_MAX` 10→11 | shared-array overflow/underflow across `struct bgp`/`peer` | G1 + G2 + G5 | build clean; `make check` 0 fail; all `bgp_*` topotests green |
| C1.2 IANA map 241 | capability not negotiated / mis-mapped | G2 unit + G4 T2 | `safi_int2iana`/`iana2int` round-trip asserts pass; T2 shows AF advertised+received |
| C1.3 `safi2str` | wrong CLI token / `show` label | G2 + G8 | `safi2str(SAFI_CRYPTO_ROUTES)=="crypto-routes"`; `show bgp ipv4 crypto-routes` accepted |
| C1.4 / C5.1–5.3 CLI nodes | can't configure AF; vtysh navigation broken | G4 T10, T11 + G8 | stanza parses, persists, round-trips; rejected in vrf instance; `exit-address-family` works |
| C2.1 `enum bgp_af_index` +2 | update-group array off-by-one → crash/corruption | G1 + G2 `afindex()` asserts + G3 ASan + G4 T3 | asserts pass; no ASan; routes propagate via update-groups |
| C2.1 `peer_afi_active_nego`/`peer_group_af_configured` | peer-group AF inheritance broken; session not counted active | G5 (`bgp_peer_group*` topotests) + G4 | no regression; crypto-routes-only peer treated as active |
| C2.2 maxpaths pin | ECMP semantics in an informational RIB; stray config line | G4 T10 + G8 | `maximum-paths` rejected with message; not emitted in `show run` |
| C3.1 capability display | `show bgp neighbor` missing/incorrect AF line | G4 T2 + G8 | capability line shows "crypto-routes" / "Crypto-Routes" |
| C3.2 nexthop encode (4 arms) | malformed MP_REACH → peer session reset / NLRI misparse downstream | G4 T3, T9 + G5 | prefix + correct nexthop received on r3 (v4 and v6); no session flaps |
| C3.2 `nh_afi` ENHE | v4-prefix-over-v6-nexthop peering fails | G4 (optional ENHE variant) | if tested: v4 crypto-route with v6 nexthop propagates |
| C3.3 `bgp_nlri_parse` dispatch | inbound UPDATE dropped or misrouted | G4 T3, T4, T8 | prefixes appear/withdraw correctly; filtered prefix absent |
| C4.1 `bgp_rd_from_dest` | wrong RD handling / crash on table walk | G1 + G4 T3 (`show` walks table) | build clean; `show bgp ipv4 crypto-routes` renders |
| C4 (no FIB gate change) | crypto-route leaks into kernel FIB | **G4 T7** (explicit) | `show ip route <pfx>` has no BGP crypto-routes entry on r2/r3; `ip route` in netns clean |
| C4.2/C4.3 rfapi arms | `--enable-bgp-vnc` build breaks | G1 with `--enable-bgp-vnc` | VNC build clean |
| C6 `show` grammar | command not found / wrong table | G4 T3, T6 + G8 | `show bgp ipv4 crypto-routes [pfx] [json]` returns the crypto-routes RIB only |
| C7 debug | no debug output / not written back | G8 | `debug bgp crypto-routes` toggles; appears in `show run` |
| C8 docs | doc build fails | G7 | `make -C doc html` clean; page linked |
| C9 tests | — | self | new unit asserts + topotest committed and green |
| **Global:** `SAFI_MAX` bump | latent regression in any AF | G5 + G6 | full `bgp_*`/`evpn_*`/`bgp_gr_*` green; no new leak |

---

## 5. Integration Topotest — `tests/topotests/bgp_crypto_routes/`

### 5.1 Topology

```
         eBGP                    eBGP
 r1 ───────────────── r2 ───────────────── r3
 AS65001             AS65002             AS65003
 - originates        - transit           - receiver
   crypto-routes:    - inbound RM on       - verifies RIB isolation
   192.0.2.0/24        r1 side:             - verifies NO FIB install
   198.51.100.0/24      drop 198.51.100/24  - also originates 192.0.2.0/24
   2001:db8:cr::/48     set large-comm        in *unicast* (isolation check)
```

Link addressing: r1–r2 `10.12.0.0/24` + `2001:db8:12::/64`; r2–r3 `10.23.0.0/24` + `2001:db8:23::/64`.

### 5.2 `r1/bgpd.conf` (sketch)

```
router bgp 65001
 neighbor 10.12.0.2 remote-as 65002
 neighbor 2001:db8:12::2 remote-as 65002
 address-family ipv4 crypto-routes
  neighbor 10.12.0.2 activate
  network 192.0.2.0/24
  network 198.51.100.0/24
 exit-address-family
 address-family ipv6 crypto-routes
  neighbor 2001:db8:12::2 activate
  network 2001:db8:cr::/48
 exit-address-family
```

### 5.3 Assertions (`test_bgp_crypto_routes.py`, via `verify_*` / `json_cmp`)

| ID | Step | Pass criterion |
|---|---|---|
| **T1** | sessions establish | `show bgp summary json` — all neighbors `Established` (v4 + v6) |
| **T2** | capability negotiated | r1 `show bgp neighbor 10.12.0.2 json`: crypto-routes AF listed as **advertised and received**; likewise r2↔r3 |
| **T3** | e2e advertise | r3 `show bgp ipv4 crypto-routes json` contains `192.0.2.0/24`, `aspath "65001 65002"`, nexthop resolvable via r2 |
| **T4** | inbound policy isolation | `198.51.100.0/24` (dropped by r2 inbound RM) **absent** from r2 and r3 crypto-routes RIB |
| **T5** | metadata via community | large-community set by r2 present in r3 `show bgp ipv4 crypto-routes 192.0.2.0/24 json` |
| **T6** | RIB isolation | on r3, `192.0.2.0/24` exists **independently** in `show bgp ipv4 unicast json` (self-originated) and `show bgp ipv4 crypto-routes json` (from r1); withdrawing one leaves the other |
| **T7** | **NO FIB install** | on r2 **and** r3: `show ip route 192.0.2.0/24 json` has no `protocol":"bgp"` entry sourced from crypto-routes; `ip -j route get 192.0.2.1` in the netns unaffected by crypto-routes |
| **T8** | withdraw | after `no network 192.0.2.0/24` on r1 → r3 crypto-routes RIB no longer has it (MP_UNREACH), within hold time |
| **T9** | IPv6 parity | T3 + T7 + T8 repeated for `2001:db8:cr::/48` |
| **T10** | config round-trip | r1 `show running-config` contains both `address-family … crypto-routes` stanzas with `network` + `neighbor activate`; after `clear bgp *` (or bgpd restart) config and RIB restore |
| **T11** | vrf rejection | `router bgp 65001 vrf RED` → `address-family ipv4 crypto-routes` → CLI rejects ("Only Unicast/Multicast/EVPN … in non-core instances") |
| **T12** | maxpaths fixed | `address-family ipv4 crypto-routes` → `maximum-paths 4` → rejected; `show run` never shows `maximum-paths` under the AF |
| **T13** | no cross-talk | existing r1↔r2 unicast session (add one) unaffected: unicast prefixes propagate normally with crypto-routes active |

### 5.4 Regression selection (G5)

Mandatory full runs: `bgp_peer_group*`, `bgp_multi_vrf_topo1`, `bgp_l3vpn_to_bgp_vrf`, `bgp_flowspec`, `bgp_unreachability`, `bgp_gr_functionality_topo2`, `bgp_vpnv4_*`, `evpn_type5_test_topo1`, `bgp_addpath*`, `bgp_dynamic_capability*`.
Rationale: these exercise `enum bgp_af_index`, `afindex()`, peer-group AF inheritance, capability negotiation, and per-AF arrays — the shared surfaces the `SAFI_MAX`/`BGP_AF_MAX` bump touches.

---

## 6. Negative & Boundary Testing

| Case | Method | Expected |
|---|---|---|
| MP_REACH with SAFI 241 but 0-length nexthop | `test_mp_attr` crafted blob | `bgp_mp_reach_parse` → error → NOTIFICATION (no crash) |
| NLRI prefixlen > 32 (v4) / > 128 (v6) | `test_mp_attr` / scapy in topo | rejected by `bgp_nlri_parse_ip` bounds check |
| Peer sends crypto-routes NLRI without capability negotiated | scapy-crafted UPDATE in topo | UPDATE treated per existing "AF not negotiated" handling; session policy unchanged |
| `neighbor X activate` in crypto-routes for a peer that doesn't support it | topo | local RIB/adj-out built; capability simply not matched; no crash |
| `SAFI_MAX`-sized stack/static array assumed == 10 somewhere | full `make check` + ASan topo | none (all arrays `[SAFI_MAX]`; `SAFI_UNREACH` did the identical 9→10 bump) |
| `afindex(AFI_L2VPN, SAFI_CRYPTO_ROUTES)` | unit assert | `== BGP_AF_MAX` (not a valid combo) |
| Config: `address-family ipv4 crypto-routes` then `network` a malformed prefix | vtysh | rejected by existing `network` validation |
| Downgrade: bgpd with the change peers with stock bgpd | manual/topo | stock side ignores the unknown capability; unicast/other AFs unaffected |

---

## 7. Performance / Scale Sanity (lightweight)

Not a v1 gate, but record:

- `struct peer` size delta from `SAFI_MAX` +1: measure `sizeof(struct peer)` before/after (`gdb -batch -ex 'p sizeof(struct peer)'`); expect +~1–2 KB. Note in PR.
- Inject 10k crypto-routes prefixes (sharpd or `network` loop) on r1; confirm r3 converges and `show bgp ipv4 crypto-routes` count matches; `top`/`show memory bgpd` sane.
- Confirm empty crypto-routes RIB (AF never activated) costs one empty `route_table` per `struct bgp` per AFI — already the case for every other SAFI.

---

## 8. Sign-off Checklist

Merge-ready when **all** true:

- [ ] G0: `git clang-format` + `checkpatch.pl` clean on every commit; each commit `Signed-off-by`; subjects `dir: …` ≤50.
- [ ] G1: `make` clean, gcc + clang, `-Werror`, with and without `--enable-bgp-vnc`, `--enable-grpc`.
- [ ] G2: `make check` 0 failures; new asserts present and passing.
- [ ] G3: ASan build of `bgp_crypto_routes` topotest — no reports.
- [ ] G4: T1–T13 pass.
- [ ] G5: mandatory regression list (§5.4) green; no new failures vs. recorded baseline.
- [ ] G6: `--valgrind-memleaks` on new suite — no bgpd leak (esp. `bgp->rib[*][SAFI_CRYPTO_ROUTES]` freed at shutdown).
- [ ] G7: `make -C doc html` clean; `crypto-routes.rst` linked from `bgp.rst`.
- [ ] G8: manual CLI/`show`/`debug`/`vtysh` walk — reviewer sign-off.
- [ ] G9: structured-reasoning template (§3) completed for all 9 change groups — reviewer sign-off.
- [ ] Every row in §4 traceability matrix has a passing test/gate.
- [ ] Bisectability: `git rebase --exec 'make -j$(nproc)' origin/master` succeeds across the series.

---

## 9. Baseline Capture (do first, before any code change)

```console
cd frr && git rev-parse HEAD > /tmp/cr_baseline_sha
./bootstrap.sh && ./configure --enable-dev-build --disable-doc && make -j"$(nproc)"
make check 2>&1 | tee /tmp/cr_baseline_makecheck.log
gdb -batch -ex 'p sizeof(struct peer)' -ex 'p sizeof(struct bgp)' bgpd/bgpd | tee /tmp/cr_baseline_sizes.txt
cd tests/topotests
sudo -E pytest -s -nauto --dist=loadfile bgp_ evpn_ 2>&1 | tee /tmp/cr_baseline_topo.log
```

The G5 gate compares against `/tmp/cr_baseline_topo.log` — only *new* failures block.
