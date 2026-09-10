# Reference Execution & Testing Plan — `crypto_routes` SAFI (FRRouting)

**Status:** Draft v1 · **Date:** 2026-09-10 · **Target:** FRR `master` (`10.8.0-dev`, submodule `bc8971168b`)
**Reads with:** [`HLD_v1.md`](./HLD_v1.md) · [`LLD_v1.md`](./LLD_v1.md) · [`REPO_ANALYSIS_LOG.md`](./REPO_ANALYSIS_LOG.md)

This plan is the end-to-end path from a clean checkout to a merge-ready `crypto_routes` change: environment → build → static gates → unit → integration (topotest) → manual interop smoke → CI → acceptance sign-off → rollback.

---

## 0. Test Strategy at a Glance

| Layer | Tool | What it proves | Root? | Runtime |
|---|---|---|---|---|
| Static | `git clang-format`, `scripts/checkpatch.pl` | Style/DCO gates that CI enforces | no | seconds |
| Build | `configure && make -Werror` | Every `switch (safi)` arm added; no warnings | no | ~10–25 min |
| Unit | `make check` (TAP) | AFI/SAFI mapping, MP-attr framing, `afindex()` | no | ~1–2 min |
| Integration | `tests/topotests/bgp_crypto_routes/` (pytest+micronet) | Capability negotiation, advertise/receive, policy, withdraw, **no FIB install** | **yes** | ~30–90 s |
| Regression | full `bgp_*` topotests | No collateral damage to other AFs | yes | ~1–3 h |
| Manual | 3× FRR in netns / topotest shell | Interop feel, `show`/`debug` UX | yes | manual |

---

## 1. Environment Setup (once)

Reference distro: **Ubuntu 24.04** (matches FRR CI + topotests). Full dependency list and rationale in [`REPO_ANALYSIS_LOG.md` §8](./REPO_ANALYSIS_LOG.md).

```console
# 1. Build deps
sudo apt-get update
sudo apt-get install -y \
   git autoconf automake libtool make libreadline-dev texinfo \
   pkg-config libpam0g-dev libjson-c-dev bison flex \
   libc-ares-dev python3-dev python3-sphinx \
   install-info build-essential libsnmp-dev perl \
   libcap-dev libelf-dev libunwind-dev \
   protobuf-c-compiler libprotobuf-c-dev

# 2. libyang from source (distro pkg usually too old)
sudo apt-get install -y cmake libpcre2-dev
git clone https://github.com/CESNET/libyang.git && cd libyang
git checkout v2.1.148
mkdir build && cd build
cmake -D CMAKE_INSTALL_PREFIX:PATH=/usr -D CMAKE_BUILD_TYPE:String=Release ..
make -j"$(nproc)" && sudo make install && sudo ldconfig
cd ../..

# 3. Topotest deps (integration layer)
sudo apt-get install -y gdb iproute2 net-tools python3-pip iputils-ping \
   iptables tshark valgrind ssmping
python3 -m pip install --user wheel
python3 -m pip install --user 'pytest>=8.3.2' 'pytest-asyncio>=0.24.0' \
   'pytest-xdist>=3.6.1' 'scapy>=2.4.5' 'libyang<4' pyyaml xmltodict 'protobuf<4'
python3 -m pip install --user \
   git+https://github.com/Exa-Networks/exabgp@0659057837cd6c6351579e9f0fa47e9fb7de7311
sudo useradd -d /var/run/exabgp/ -s /bin/false exabgp || true

# 4. FRR runtime user/groups (only needed to run daemons / topotests)
sudo groupadd -r -g 92 frr;  sudo groupadd -r -g 85 frrvty
sudo adduser --system --ingroup frr --home /var/run/frr/ \
   --gecos "FRR suite" --shell /sbin/nologin frr
sudo usermod -a -G frrvty frr
```

---

## 2. Build

From the repo (`AI_FRR/`), the FRR tree is the `frr/` submodule:

```console
cd frr
git submodule update --init --recursive     # pceplib etc.
./bootstrap.sh

# Developer build: extra asserts + ASan, run from build tree, no install needed for unit tests
./configure \
    --enable-dev-build \
    --enable-address-sanitizer \
    --prefix=/usr --localstatedir=/var --sysconfdir=/etc \
    --enable-multipath=64 \
    --disable-doc          # re-enable for the doc commit

make -j"$(nproc)"
```

**Expected first-pass failures = the checklist.** With `-Werror` + `-Wswitch`, the build stops at each `switch (safi)` that lacks `case SAFI_CRYPTO_ROUTES:`. Fix, rebuild, repeat. Enumerate them up front:

```console
grep -rn "case SAFI_UNREACH:" bgpd/ lib/
grep -rn "switch (safi)\|switch (packet->safi)\|switch (table->safi)\|switch (bgp_dest" bgpd/ lib/
```

**Targeted compile during dev (faster loop):**

```console
make -j"$(nproc)" bgpd/bgpd          # just the daemon
make -j"$(nproc)" lib/libfrr.la      # just the library
```

For a topotest-capable build:

```console
./configure --prefix=/usr --sbindir=/usr/lib/frr --libdir=/usr/lib/frr \
    --sysconfdir=/etc --localstatedir=/var --enable-multipath=64 \
    --enable-user=frr --enable-group=frr --enable-vty-group=frrvty
make -j"$(nproc)" && sudo make install
sudo ln -sf /usr/lib/frr/vtysh /usr/bin/vtysh
```

---

## 3. Static Review Gates (must pass — CI enforces)

```console
# from frr/
git clang-format origin/master          # or: git clang-format HEAD~10
scripts/checkpatch.pl --no-tree -g origin/master..HEAD    # per-commit
# spot-check a file context:
scripts/checkpatch.pl --no-tree -f bgpd/bgp_vty.c

# DCO: every commit has Signed-off-by
git log origin/master..HEAD --format='%h %s%n%b' | grep -c 'Signed-off-by'
```

Checklist (from [LLD §7](./LLD_v1.md#7-pre-compilation-review-checklist)):
- SPDX `GPL-2.0-or-later` header on any new `.c/.h/.py`; `<zebra.h>` first include.
- No `strcpy`/`strcat`/`sprintf`; `sizeof()` in size args.
- New lists/hashes use `lib/typesafe.h`.
- Commit subjects `dir: imperative summary`, ≤ 50 chars, body wrapped 72.

---

## 4. Unit Tests (`make check`)

### 4.1 Run existing suite (regression)

```console
make check 2>&1 | tee /tmp/make-check.log
# TAP summary at the end; 0 failures required.
# AFI/SAFI-relevant binaries:
./tests/bgpd/test_mp_attr        # MP_REACH/MP_UNREACH parse
./tests/bgpd/test_capability     # MP capability parse
./tests/bgpd/test_aspath ./tests/bgpd/test_attr_parse
./tests/lib/test_prefix2str
```

### 4.2 New / extended unit coverage

| Test | File | Assertions |
|---|---|---|
| SAFI mapping round-trip | extend `tests/bgpd/test_capability.c` or new `tests/lib/test_afi_safi.c` | `safi_int2iana(SAFI_CRYPTO_ROUTES) == 241`; `safi_iana2int(241) == SAFI_CRYPTO_ROUTES`; `safi2str(SAFI_CRYPTO_ROUTES) == "crypto-routes"`; `bgp_map_afi_safi_iana2int(1,241,&a,&s)` → `(AFI_IP, SAFI_CRYPTO_ROUTES)`, returns 0. |
| MP capability for new AF | extend `tests/bgpd/test_capability.c` | feed a crafted MP capability TLV with AFI=1/SAFI=241 → parser accepts, sets the expected `afc_recv` bit. |
| MP_REACH NLRI for new SAFI | extend `tests/bgpd/test_mp_attr.c` | a `MP_REACH_NLRI` blob with AFI=1/SAFI=241 + one prefix decodes via `bgp_nlri_parse_ip` path without error; prefix count == 1. |
| `afindex()` | small `tests/bgpd/test_*` or assert in existing | `afindex(AFI_IP, SAFI_CRYPTO_ROUTES) == BGP_AF_IPV4_CRYPTO_ROUTES`; `afindex(AFI_L2VPN, SAFI_CRYPTO_ROUTES) == BGP_AF_MAX`. |

Wire new `.c` tests into `tests/subdir.am` (`check_PROGRAMS`, `TESTS`, `_SOURCES`, `_LDADD = lib/libfrr.la $(LIBCAP)`) with a sibling `test_*.py` frrtest wrapper — follow any existing `tests/bgpd/test_*` pair.

```console
make check TESTS='tests/bgpd/test_mp_attr tests/bgpd/test_capability'
```

---

## 5. Integration Test — `tests/topotests/bgp_crypto_routes/`

Model on `tests/topotests/bgp_unreachability/` (commit `617982f55b`) and `tests/topotests/bgp_l3vpn_to_bgp_vrf/`.

### 5.1 Topology

```
      eBGP (AS65001)        eBGP (AS65002)
 r1 ─────────────────── r2 ─────────────────── r3
 AS65001               AS65002               AS65003
 originates            transit +             receiver +
 crypto-routes         inbound RM filter     verifies no FIB
```

- `r1`: `address-family ipv4/ipv6 crypto-routes`, `network 192.0.2.0/24`, `network 2001:db8:cr::/48`, `neighbor r2 activate`.
- `r2`: activate to both neighbors; **inbound route-map on r1 side dropping `198.51.100.0/24`** (to prove policy isolation) and setting a large-community.
- `r3`: activate to r2; also has plain `address-family ipv4 unicast` with the *same* prefix `192.0.2.0/24` originated in unicast (to prove RIB isolation).

### 5.2 `test_bgp_crypto_routes.py` — assertions

| # | Step | Assertion (via `vtysh -c "… json"` + `topotest.json_cmp`) |
|---|---|---|
| T1 | Sessions up | `show bgp summary json` — all neighbors `Established`. |
| T2 | Capability negotiated | `show bgp neighbor <r2> json` on r1 → `.[].neighborCapabilities.multiprotocolExtensions` (or equivalent) lists `ipv4CryptoRoutes` advertised **and** received. |
| T3 | Advertise/receive | `show bgp ipv4 crypto-routes json` on r3 → contains `192.0.2.0/24` with nexthop chain via r2, path `65001 65002` … (AS_PATH check). |
| T4 | Policy isolation | `198.51.100.0/24` (dropped by r2's inbound RM) is **absent** from r3's crypto-routes table; present nowhere downstream. |
| T5 | Metadata | large-community set by r2 visible in `show bgp ipv4 crypto-routes 192.0.2.0/24 json` on r3. |
| T6 | **RIB isolation** | On r3, `show bgp ipv4 unicast 192.0.2.0/24 json` (locally originated) and `show bgp ipv4 crypto-routes 192.0.2.0/24 json` (from r1) are **independent** entries; changing one policy doesn't move the other. |
| T7 | **No FIB install** | On r2 and r3: `show ip route 192.0.2.0/24 json` shows **only** the unicast/none source — **no** BGP crypto-routes entry in zebra. `ip route show` in the namespace confirms. |
| T8 | Withdraw | `no network 192.0.2.0/24` on r1 → within hold, `show bgp ipv4 crypto-routes json` on r3 no longer has it (MP_UNREACH propagated). |
| T9 | IPv6 parity | repeat T3/T7/T8 for `2001:db8:cr::/48`. |
| T10 | Config round-trip | `show running-config` on r1 contains the `address-family ipv4 crypto-routes` stanza with `network` + `neighbor activate`; reload (`clear bgp *` / daemon restart) preserves it. |
| T11 | Non-default instance rejected | `router bgp 65001 vrf RED` + `address-family ipv4 crypto-routes` → CLI rejects (core-instance only). |

### 5.3 Run

```console
cd tests/topotests/bgp_crypto_routes
sudo -E python3 -m pytest -s -v ./test_bgp_crypto_routes.py
# logs: /tmp/topotests/…   ;  analyze: ../analyze.py -Ar run-save
```

### 5.4 Regression sweep (pre-merge)

```console
cd tests/topotests
sudo -E pytest -s -nauto --dist=loadfile bgp_          # all BGP suites
# and at minimum, these capability/AF-sensitive ones fully:
sudo -E pytest bgp_multi_vrf_topo1 bgp_l3vpn_to_bgp_vrf bgp_flowspec \
               bgp_unreachability evpn_type5_test_topo1 bgp_gr_functionality_topo2
```

### 5.5 Sanitizer / leak run

```console
# build with --enable-address-sanitizer (see §2), then:
cd tests/topotests/bgp_crypto_routes
sudo -E pytest -s -v --asan-abort ./test_bgp_crypto_routes.py
# optional valgrind leak check:
sudo -E pytest -s -v --valgrind-memleaks ./test_bgp_crypto_routes.py
```
Expect: no ASan reports, no new `Direct/Indirect leak` from bgpd on shutdown (the new RIB tables must be freed by the existing `bgp_table_finish` loop in `bgp_free()` — verify no leak from `bgp->rib[*][SAFI_CRYPTO_ROUTES]`).

---

## 6. Manual Interop / UX Smoke Test

Fast loop without full topotest, using the topotest micronet shell **or** three Linux netns:

```console
# using the topotest interactive shell:
cd tests/topotests/bgp_crypto_routes
sudo -E pytest -s --pause ./test_bgp_crypto_routes.py     # drops to a shell with the topo up
#   r1# vtysh
#   r1# configure terminal
#   r1(config)# router bgp 65001
#   r1(config-router)# address-family ipv4 crypto-routes
#   r1(config-router-af)# network 203.0.113.0/24
#   r1(config-router-af)# end
#   r1# show bgp ipv4 crypto-routes
#   r1# debug bgp crypto-routes
#   r1# show bgp neighbor 10.0.0.2   (check capability line)
#   r3# show bgp ipv4 crypto-routes 203.0.113.0/24
#   r3# show ip route 203.0.113.0/24        <-- must NOT be from BGP crypto-routes
```

UX checklist:
- `?` completion shows `crypto-routes` under `address-family ipv4`/`ipv6` and under `show bgp ipv4`.
- `show running-config` places the stanza consistently (indent, order) with other AFs.
- `debug bgp crypto-routes` emits on UPDATE in/out and is written back by `show running-config`.
- `vtysh` navigation: `exit-address-family` returns to `config-router`; `end` to enable.

### Optional: second-implementation interop (defer to v2)
Because v1 uses **IANA SAFI 241 (private use)**, cross-vendor interop is only meaningful if the other side is configured for the same private value. Note in the PR; not a v1 gate.

---

## 7. CI Expectations (GitHub Actions `github-ci`)

| Job | Expectation |
|---|---|
| `clang-format` / `checkpatch` / `DCO` | green (covered by §3) |
| `build-*` matrix (Ubuntu/Debian/Alpine/FreeBSD, gcc+clang) | green; watch **clang `-Wswitch`** — stricter than gcc on enum switches |
| `Addresssanitizer` topotest job | new suite green, no ASan |
| `Topotests part N` | new suite assigned to a shard; full `bgp_*` green |
| `Static analyzer` (Coverity via nightly) | no new defects (the `SAFI_UNREACH` series had a Coverity follow-up `e1d6c771b8` — budget one) |

Local pre-push equivalent:

```console
cd frr
./bootstrap.sh && ./configure --enable-dev-build --disable-doc && make -j"$(nproc)"
make check
( cd tests/topotests && sudo -E pytest -s -nauto --dist=loadfile bgp_crypto_routes bgp_unreachability bgp_flowspec )
```

---

## 8. Acceptance Criteria Matrix

| ID | Criterion | Verified by |
|---|---|---|
| A1 | Clean build, no new warnings, gcc **and** clang | §2, §7 |
| A2 | `make check` 0 failures incl. new unit tests | §4 |
| A3 | Capability `MP(1|2, 241)` advertised + received | T2 |
| A4 | Prefix propagates r1→r2→r3 in crypto-routes AF with correct AS_PATH/nexthop | T3, T9 |
| A5 | Inbound/outbound policy applies per-AF, isolated from unicast | T4, T5, T6 |
| A6 | **Zero FIB install** — nothing in zebra/kernel from crypto-routes | T7 |
| A7 | Withdraw (`MP_UNREACH`) propagates | T8 |
| A8 | Config persists across restart / `show running-config` round-trip | T10 |
| A9 | Rejected in non-default BGP instance | T11 |
| A10 | No regression in `bgp_*` topotests | §5.4 |
| A11 | No ASan / leak on the new suite | §5.5 |
| A12 | `git clang-format` + `checkpatch` clean; every commit `Signed-off-by`; bisectable | §3, `git rebase --exec 'make -j'` |
| A13 | User doc page builds (`--enable-doc`, `make -C doc`) and is linked from `bgp.rst` | §2 doc commit |

---

## 9. Rollback / Revert

- The change is **additive and opt-in**. Disable at runtime: `no neighbor X activate` in the crypto-routes AF, or simply don't configure the stanza — zero effect on other families.
- Code revert: `git revert` the 8–10 commit range (bisectable, each reverts cleanly). The `SAFI_MAX` bump reverts with the enum commit; no data migration (RIB is in-memory only).
- No on-disk format change, no ZAPI change, no YANG change → no downgrade hazard for `zebra` or `mgmtd`.

---

## 10. Execution Timeline (reference)

| Day | Work |
|---|---|
| 1 | Env + baseline build + baseline `make check` + baseline `bgp_*` topotest run (record green set). |
| 2 | Commits 1–3 (lib enum + mapping + attr/packet + capability). Build loop against `-Wswitch`. Unit tests for mapping. |
| 3 | Commits 4–5 (route processing, af_index, RIB, maxpaths). `make check`. |
| 4 | Commit 6 (CLI + nodes + vtysh). Manual smoke via `--pause`. |
| 5 | Commits 7–8 (show + debug). Write topotest `bgp_crypto_routes/`. |
| 6 | Topotest pass + regression sweep + ASan run. Doc commit. |
| 7 | checkpatch/clang-format polish, rebase for bisectability, PR with test evidence. |

---

## Appendix — Quick Command Reference

```console
# Build (dev)
cd frr && ./bootstrap.sh && ./configure --enable-dev-build --disable-doc && make -j"$(nproc)"

# Enumerate switch sites to patch
grep -rn "case SAFI_UNREACH:" bgpd/ lib/

# Unit
make check
make check TESTS='tests/bgpd/test_mp_attr tests/bgpd/test_capability'

# Static
git clang-format origin/master
scripts/checkpatch.pl --no-tree -g origin/master..HEAD

# Integration
cd tests/topotests/bgp_crypto_routes && sudo -E pytest -s -v ./test_bgp_crypto_routes.py
cd tests/topotests && sudo -E pytest -s -nauto --dist=loadfile bgp_

# Interactive topo
sudo -E pytest -s --pause ./test_bgp_crypto_routes.py
```
