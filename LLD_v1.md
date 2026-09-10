# LLD v1 — New BGP Address Family `crypto_routes` (FRRouting)

**Status:** Draft v1 · **Date:** 2026-09-10 · **Target:** FRR `master` (`10.8.0-dev`, submodule `bc8971168b`)
**Reads with:** [`HLD_v1.md`](./HLD_v1.md) · [`REPO_ANALYSIS_LOG.md`](./REPO_ANALYSIS_LOG.md) · [`EXECUTION_TESTING_PLAN.md`](./EXECUTION_TESTING_PLAN.md)

This document is the file-, function-, and line-level implementation design. It is written to be **reviewable before compilation** (problem-statement requirement): every hunk states *what* changes and *why it is safe*. Line numbers are anchors against submodule commit `bc8971168b` and will drift; the reliable locator is **"the `switch` arm next to `case SAFI_UNREACH:`"**, since the merged `SAFI_UNREACH` series (`git log --grep UNREACH`) touched the same sites.

---

## 1. Design in One Paragraph

`crypto_routes` is a new **SAFI** (`SAFI_CRYPTO_ROUTES`) under the existing `AFI_IP` / `AFI_IP6`. Its NLRI is a plain IP prefix, wire-identical to unicast, carried in `MP_REACH_NLRI` / `MP_UNREACH_NLRI`. Therefore the receive parser, RIB (radix trie), inbound/outbound policy, best-path, and update-group code are all **the existing unicast code**, reached by adding `case SAFI_CRYPTO_ROUTES:` beside `case SAFI_UNICAST:` (or a no-op arm where a `switch` only needs to stay exhaustive for `-Wswitch`). The family is **control-plane only** — it is deliberately *excluded* from the `bgp_zebra` FIB-install gate. No new source file is required in v1.

---

## 2. Names & Constants

| Symbol | Value | File |
|---|---|---|
| `SAFI_CRYPTO_ROUTES` | `10` (new); `SAFI_MAX` → `11` | `lib/zebra.h` |
| `IANA_SAFI_CRYPTO_ROUTES` | `241` (IANA Private-Use range 241–254) | `lib/iana_afi.h` |
| `safi2str(SAFI_CRYPTO_ROUTES)` | `"crypto-routes"` | `lib/prefix.c` |
| `BGP_AF_IPV4_CRYPTO_ROUTES`, `BGP_AF_IPV6_CRYPTO_ROUTES` | before `BGP_AF_MAX` | `bgpd/bgpd.h` |
| `BGP_IPV4_CRYPTO_NODE`, `BGP_IPV6_CRYPTO_NODE` | after `BGP_IPV6U_NODE` | `lib/command.h` |
| CLI SAFI token | `crypto-routes` | `bgpd/bgp_vty.c` grammar + `bgp_vty_safi_from_str()` |
| `BGP_DEBUG_CRYPTO_ROUTES` | `0x01` | `bgpd/bgp_debug.h` |
| Config stanza | `address-family ipv4 crypto-routes` / `… ipv6 …` | `bgpd/bgp_vty.c`, `vtysh/vtysh.c` |
| Doc page | `doc/user/crypto-routes.rst` | `doc/user/` |
| Topotest dir | `tests/topotests/bgp_crypto_routes/` | `tests/` |

---

## 3. File-by-File Change Map

### 3.1 `lib/zebra.h` — the SAFI enum + NSF macro

```c
 typedef enum {
 	SAFI_UNSPEC = 0,
 	...
 	SAFI_BGP_LS = 8,
 	SAFI_UNREACH = 9,
-	SAFI_MAX = 10
+	SAFI_CRYPTO_ROUTES = 10,   /* crypto/overlay reachable prefixes, control-plane only */
+	SAFI_MAX = 11
 } safi_t;
```

`FOREACH_AFI_SAFI` iterates `SAFI_UNICAST .. < SAFI_MAX`, so the new SAFI is automatically visited by every existing loop (RIB alloc/free, capability build, GR init, counters). **Do NOT add it to `FOREACH_AFI_SAFI_NSF`** — v1 does not participate in Graceful-Restart NSF (no forwarding state to preserve), same as `LABELED_UNICAST` / `FLOWSPEC` / `BGP_LS`.

**Safety:** identical in shape to the `SAFI_UNREACH` bump (`8→9`, `9→10`) merged in commit `023da6de44`. All consumers index `[SAFI_MAX]`.

### 3.2 `lib/iana_afi.h` — wire ↔ internal mapping (4 hunks)

```c
 typedef enum {
 	...
 	IANA_SAFI_MPLS_VPN = 128,
 	IANA_SAFI_FLOWSPEC = 133,
+	IANA_SAFI_CRYPTO_ROUTES = 241,   /* IANA "Reserved for Private Use" (241-254) */
 } iana_safi_t;

 static inline safi_t safi_iana2int(iana_safi_t safi) {
 	switch (safi) {
 	...
 	case IANA_SAFI_FLOWSPEC:
 		return SAFI_FLOWSPEC;
+	case IANA_SAFI_CRYPTO_ROUTES:
+		return SAFI_CRYPTO_ROUTES;
 	...
 	}
 }

 static inline iana_safi_t safi_int2iana(safi_t safi) {
 	switch (safi) {
 	...
 	case SAFI_FLOWSPEC:
 		return IANA_SAFI_FLOWSPEC;
+	case SAFI_CRYPTO_ROUTES:
+		return IANA_SAFI_CRYPTO_ROUTES;
 	case SAFI_UNSPEC:
 	case SAFI_MAX:
 		return IANA_SAFI_RESERVED;
 	}
 }
```

This is what makes `bgp_map_afi_safi_iana2int()` / `..._int2iana()` (`bgpd/bgpd.c:1000`) work for the new SAFI — and those are called from `bgp_open.c` (capability), `bgp_attr.c` (MP_REACH/UNREACH), `bgp_packet.c`. **Once this hunk lands, capability advertise + receive + MP NLRI framing are done** — they are all generic loops keyed on the mapping + `peer->afc[][]`.

### 3.3 `lib/prefix.c` — `safi2str()`

```c
 const char *safi2str(safi_t safi) {
 	switch (safi) {
 	...
 	case SAFI_UNREACH:
 		return "unreachability";
+	case SAFI_CRYPTO_ROUTES:
+		return "crypto-routes";
 	case SAFI_UNSPEC:
 	case SAFI_MAX:
 		return "unknown";
 	}
 }
```

There is **no `switch (safi)` in `lib/zclient.c`** (it range-checks `SAFI_UNICAST .. SAFI_MAX`), and v1 sends nothing to zebra, so `lib/` changes end here (~15 lines total).

### 3.4 `bgpd/bgpd.h` — `enum bgp_af_index` + `afindex()` + container helper

```c
 enum bgp_af_index {
 	BGP_AF_START,
 	BGP_AF_IPV4_UNICAST = BGP_AF_START,
 	...
 	BGP_AF_IPV4_UNREACH,
 	BGP_AF_IPV6_UNREACH,
+	BGP_AF_IPV4_CRYPTO_ROUTES,
+	BGP_AF_IPV6_CRYPTO_ROUTES,
 	BGP_AF_MAX
 };
```

`afindex(afi, safi)` (`bgpd/bgpd.h`, static inline) — add arms in the `AFI_IP` and `AFI_IP6` inner switches, and the "not valid here" arm in `AFI_L2VPN` / `AFI_BGP_LS`:

```c
 	case AFI_IP:
 		switch (safi) {
 		...
 		case SAFI_UNREACH:
 			return BGP_AF_IPV4_UNREACH;
+		case SAFI_CRYPTO_ROUTES:
+			return BGP_AF_IPV4_CRYPTO_ROUTES;
 		case SAFI_EVPN:
 		case SAFI_UNSPEC:
 		case SAFI_MAX:
 			return BGP_AF_MAX;
 		}
 	case AFI_IP6:      /* ... symmetric: return BGP_AF_IPV6_CRYPTO_ROUTES ... */
 	case AFI_L2VPN:    /* add `case SAFI_CRYPTO_ROUTES:` to the "return BGP_AF_MAX" group */
 	case AFI_BGP_LS:   /* add `case SAFI_CRYPTO_ROUTES:` to the "return BGP_AF_MAX" group */
```

Also check the other exhaustive `switch (safi)` near `bgpd.h:3075–3138` (the `bgp_afi_safi_get_container` / valid-index helper) — add a `case SAFI_CRYPTO_ROUTES:` arm mirroring `SAFI_UNREACH`.

**Update-group arrays** `update_groups[BGP_AF_MAX]` and `peer_af_array[BGP_AF_MAX]` resize automatically.

### 3.5 `bgpd/bgpd.c` — RIB alloc + maxpaths pin + node-name plumbing

- **RIB allocation:** `bgp_create()` (`bgpd.c:4047`) already loops `FOREACH_AFI_SAFI` and calls `bgp_table_init(bgp, afi, safi)` for `rib`, `static_routes`, `aggregate`. **No change** — `bgp->rib[AFI_IP][SAFI_CRYPTO_ROUTES]` is created for free.
- **maxpaths pin** (`bgpd.c:4056`, mirror the `SAFI_UNREACH` special-case):

```c
-		if (safi == SAFI_UNREACH) {
+		if (safi == SAFI_UNREACH || safi == SAFI_CRYPTO_ROUTES) {
 			bgp_maximum_paths_set(bgp, afi, safi, BGP_PEER_EBGP, 1, 0);
 			bgp_maximum_paths_set(bgp, afi, safi, BGP_PEER_IBGP, 1, 0);
 		} else { ... }
```

- **`bgp_map_afi_safi_*`** (`bgpd.c:1000/1014`) — **no change**, they delegate to `lib/iana_afi.h` (§3.2).
- The `SAFI_UNREACH` CLI-nodes commit (`4154e91508`) also touched `bgpd.c` (~12 lines) for `bgp_afi_safi_str` / config-context plumbing — grep `bgpd.c` for `SAFI_UNREACH` and add the parallel `SAFI_CRYPTO_ROUTES` arm at each (2–3 sites, all `switch`/`if` string mappers).

### 3.6 `bgpd/bgp_open.c` — capability display arm

Advertise (`bgp_open_capability`, iterates `FOREACH_AFI_SAFI` on `peer->afc[afi][safi]`) and receive (`bgp_capability_mp` → `bgp_map_afi_safi_iana2int`) are **already generic**. Only the human-readable dump needs an arm — mirror commit `61f097d557` (17 lines). Grep `bgp_open.c` for `SAFI_UNREACH`; each hit is a `switch`/`if` producing a capability name string — add `case SAFI_CRYPTO_ROUTES:` / `"crypto-routes"` beside it. (Typically in `bgp_capability_mp` debug logging and `bgp_show_capability` helpers.)

### 3.7 `bgpd/bgp_attr.c` — MP_REACH / MP_UNREACH

`bgp_mp_reach_parse()` / `bgp_mp_unreach_parse()` (`bgp_attr.c` ~2822, ~3057) already:
- read `pkt_afi`/`pkt_safi`, call `bgp_map_afi_safi_iana2int()` (works after §3.2),
- for the nexthop + NLRI, call into `bgp_nlri_parse()`.

Add `case SAFI_CRYPTO_ROUTES:` wherever `bgp_attr.c` has an exhaustive `switch (safi)` (grep for `SAFI_UNREACH` — the `1a004a0281` commit touched `bgp_attr.c` for ~114 lines because UNREACH needed TLV attr handling; **crypto_routes needs far less** — just the "treat nexthop/NLRI like unicast" arms). Concretely:
- nexthop-length validation switch: treat like `SAFI_UNICAST` (accept 4-byte v4 / 16- or 32-byte v6 nexthop).
- any `safi == SAFI_UNICAST || safi == SAFI_MULTICAST` guard that gates generic MP handling → add `|| safi == SAFI_CRYPTO_ROUTES`.

### 3.8 `bgpd/bgp_packet.c` — NLRI dispatch

`bgp_nlri_parse()` (`bgp_packet.c:314`):

```c
 	switch (packet->safi) {
 	case SAFI_UNICAST:
 	case SAFI_MULTICAST:
+	case SAFI_CRYPTO_ROUTES:
 		return bgp_nlri_parse_ip(peer, mp_withdraw ? NULL : attr, packet);
 	...
 	}
```

**This one line is the entire receive-path NLRI implementation.** `bgp_nlri_parse_ip()` already does prefix decode, bounds checking, `bgp_update()` / `bgp_withdraw()` with the packet's `afi`/`safi`.

Other `bgp_packet.c` switches (EoR handling, route-refresh, `bgp_map_afi_safi_int2iana` call sites at lines 193/1151/…) are covered because they either take `default` or iterate generically. Grep `SAFI_UNREACH` in `bgp_packet.c` (commit `1a004a0281` touched 3 lines) and mirror.

### 3.9 `bgpd/bgp_route.c` + `bgp_route.h` — policy, best-path, show, **FIB gate**

| Site (anchor) | Change |
|---|---|
| `bgp_nlri_parse_ip()` prefix family check | Accepts any prefix for `afi`; already SAFI-agnostic → no change. |
| `bgp_update()` / `bgp_withdraw()` | Take `(afi, safi)` args → no change; inbound `filter[afi][safi]` already applied. |
| `bgp_best_selection()` / `bgp_process_main_one()` | Generic → no change to selection. |
| **zebra announce gate** (`bgp_route.c` ~4547, ~4589): `if ((afi==IP||IP6) && safi==SAFI_UNICAST)` | **Leave as-is.** crypto_routes falls through → **never installed to FIB**. This is the deliberate v1 boundary. |
| `subgroup_announce_check()` (`bgp_route.c:2434`) — outbound eligibility | **No change.** It is SAFI-generic (`filter[afi][safi]`, per-AF flags); the SAFI-specific arms (`SAFI_LABELED_UNICAST` label check, `SAFI_MPLS_VPN` leak guard, SRv6) are opt-in `if` blocks that crypto_routes simply doesn't enter. Outbound advertisement therefore works as-is. |
| `bgp_update_martian_nexthop()` (`bgp_route.c:5765`): validates only `SAFI_UNICAST`/`MULTICAST`/`EVPN` | **No change** — it already returns "valid" (skips the martian check) for any other SAFI, which is the desired behaviour for an informational nexthop. |
| Any *exhaustive* `switch (safi)` in `bgp_route.c` / `bgp_route.h` (grep `SAFI_UNREACH`) | Add `case SAFI_CRYPTO_ROUTES:` mirroring `SAFI_UNICAST` behaviour, only where `-Wswitch` requires it. |
| `bgp_show()` / `bgp_show_table()` | Generic table walk → no change; reached via the new `show` command (§3.12). |
| `bgp_static_set()` / `bgp_static_unset()` (the `network` cmd) | SAFI-generic → no change; only needs `install_element` on the new nodes (§3.11). |
| `bgp_route.h` | No new struct needed (crypto_routes has no per-path extra state, unlike `struct bgp_path_info_extra_unreach`). Add `case`/prototype only if an exhaustive switch in a header inline demands it. |

### 3.10 `bgpd/bgp_nht.c`, `bgp_fsm.c`, `bgp_bmp.c`, `bgp_updgrp_packet.c`, `bgp_mplsvpn.c`, `bgp_label.c`, `bgp_rpki.c`, `bgp_mac.c`, `bgp_ls.c`, `bgp_evpn*.c`, `bgp_flowspec.c`, `rfapi/*`

These appear in the `SAFI_UNREACH` diffstat only because of **`-Wswitch` exhaustiveness** on `enum safi_t`. For each, grep the file for `case SAFI_UNREACH:` (or `SAFI_BGP_LS`) and add `case SAFI_CRYPTO_ROUTES:` to the **same arm** (almost always the "not-applicable / break / return default" group). Expected: 1–3 lines per file, ~10 files, **zero behavioural change** in those subsystems.

`bgpd/rfapi/rfapi_import.c` + `rfapi_monitor.c`: exactly mirror commit `023da6de44` — add `case SAFI_CRYPTO_ROUTES:` to the `case SAFI_BGP_LS: case SAFI_UNSPEC: …` fall-through groups (VNC is `--enable-bgp-vnc`, still must compile).

### 3.11 `lib/command.h` + `bgpd/bgp_vty.c` + `bgpd/bgp_vty.h` — CLI nodes & grammar

**`lib/command.h`** (mirror `BGP_IPV4U_NODE`, commit `4154e91508`):

```c
 	BGP_IPV4U_NODE,
 	BGP_IPV6U_NODE,
+	BGP_IPV4_CRYPTO_NODE,   /* BGP IPv4 crypto-routes address family */
+	BGP_IPV6_CRYPTO_NODE,   /* BGP IPv6 crypto-routes address family */
```
(Place before the node used as an upper bound if any; keep contiguous with the other BGP nodes.)

**`bgpd/bgp_vty.h`** — extend the grammar macro:

```c
 #define BGP_SAFI_WITH_LABEL_CMD_STR \
-	"<unicast|multicast|vpn|labeled-unicast|flowspec|unreachability>"
+	"<unicast|multicast|vpn|labeled-unicast|flowspec|unreachability|crypto-routes>"
 #define BGP_SAFI_WITH_LABEL_HELP_STR \
 	BGP_AF_MODIFIER_STR ... (add one more BGP_AF_MODIFIER_STR)
```

**`bgpd/bgp_vty.c`** — add `SAFI_CRYPTO_ROUTES` arms to each helper (all have a `SAFI_UNREACH` line today):

| Function | Add |
|---|---|
| `bgp_node_type(afi, safi)` (~172) | `case SAFI_CRYPTO_ROUTES: return BGP_IPV4_CRYPTO_NODE;` (and IPv6) |
| `bgp_node_afi(vty)` (~463) | `case BGP_IPV6_CRYPTO_NODE:` → `AFI_IP6` |
| `bgp_node_safi(vty)` (~490) | `case BGP_IPV4_CRYPTO_NODE: case BGP_IPV6_CRYPTO_NODE: safi = SAFI_CRYPTO_ROUTES;` |
| `bgp_vty_safi_from_str()` (~569) | `else if (strmatch(safi_str, "crypto-routes")) safi = SAFI_CRYPTO_ROUTES;` |
| `bgp_vty_afi_safi_str()` / `get_afi_safi_vty_str()` / `get_afi_safi_json_str()` (~640–720) | `"IPv4 Crypto-Routes"` / `"ipv4-crypto-routes"` arms |
| `address_family_ipv4_safi_cmd` / `_ipv6_safi_cmd` DEFUN (~11764) | grammar already widened via the macro; `bgp_node_type()` handles routing. Non-default-instance guard: leave crypto-routes out of the allowed set (core-instance only, like flowspec). |
| `bgp_config_write_family()` frame (~22149) | `else if (safi == SAFI_CRYPTO_ROUTES) vty_frame(vty, "ipv4 crypto-routes");` (and v6) |
| `bgp_config_write()` (~22855) | `bgp_config_write_family(vty, bgp, AFI_IP, SAFI_CRYPTO_ROUTES);` (and v6) |
| `cmd_node` structs (~23002) | add `bgp_ipv4_crypto_node` / `bgp_ipv6_crypto_node` (`.node`, `.parent_node = BGP_NODE`, `.prompt = "%s(config-router-af)# "`) |
| `bgp_vty_init()` (~23481) | `install_node(&bgp_ipv4_crypto_node); install_default(BGP_IPV4_CRYPTO_NODE);` (and v6) |
| `install_element(BGP_IPV4_CRYPTO_NODE, …)` | `&neighbor_activate_cmd`, `&no_neighbor_activate_cmd`, `&neighbor_route_map_cmd`(+no), `&neighbor_prefix_list_cmd`(+no), `&neighbor_filter_list_cmd`(+no), `&neighbor_distribute_list_cmd`(+no), `&neighbor_maximum_prefix_*`, `&bgp_network_cmd`/`&no_bgp_network_cmd`, `&bgp_table_map_cmd`, `&exit_address_family_cmd`. Copy the `BGP_IPV4U_NODE` block and s/U_NODE/_CRYPTO_NODE/. |

**`bgpd/bgp_debug.c` node membership** — add `BGP_IPV4_CRYPTO_NODE`/`BGP_IPV6_CRYPTO_NODE` to the `vty->node == BGP_IPV4U_NODE || …` checks (~12120) that recognise an AF sub-node.

### 3.12 `bgpd/bgp_vty.c` / `bgp_route.c` — `show` commands

Mirror commit `875172fd44` but simpler (no custom formatter — the prefix table prints like unicast):

```
show [ip] bgp [<view|vrf> NAME] ipv4 crypto-routes [<A.B.C.D/M>] [detail] [json]
show [ip] bgp [<view|vrf> NAME] ipv6 crypto-routes [<X:X::X:X/M>] [detail] [json]
show [ip] bgp [<view|vrf> NAME] ipv4 crypto-routes neighbors <A.B.C.D> [advertised-routes|received-routes] [json]
```

Implementation: the existing `show bgp <afi> <safi> …` DEFPYs already accept an `safi` argv token via `bgp_vty_safi_from_str()`. Adding `crypto-routes` to the grammar string of the generic `show bgp` family commands (search `"unreachability"` in the DEFPY strings at ~12214, ~15298, ~18977) routes them through `bgp_show()` with `SAFI_CRYPTO_ROUTES`. Verify `bgp_show()` / `bgp_show_table()` have no `assert(safi == …)` that excludes it (they don't — they're generic).

`show bgp summary` per-AF: `BGP_AF_IPV4_CRYPTO_ROUTES` in `enum bgp_af_index` + `get_afi_safi_vty_str()` arm gives the summary line for free.

### 3.13 `bgpd/bgp_debug.c` + `bgp_debug.h` — `debug bgp crypto-routes`

Mirror commit `c3ffb5705e` (37 lines):

```c
/* bgp_debug.h */
extern unsigned long conf_bgp_debug_crypto_routes;
extern unsigned long term_bgp_debug_crypto_routes;
#define BGP_DEBUG_CRYPTO_ROUTES 0x01

/* bgp_debug.c */
unsigned long conf_bgp_debug_crypto_routes;
unsigned long term_bgp_debug_crypto_routes;

DEFPY(debug_bgp_crypto_routes, debug_bgp_crypto_routes_cmd,
      "[no$no] debug bgp crypto-routes",
      NO_STR DEBUG_STR BGP_STR "BGP crypto-routes (SAFI_CRYPTO_ROUTES) debugging\n")
{ /* DEBUG_ON/OFF(crypto_routes, CRYPTO_ROUTES) — copy the unreachability DEFPY */ }
```
+ `bgp_debug_init()` `install_element(ENABLE_NODE/CONFIG_NODE, &debug_bgp_crypto_routes_cmd);`
+ `no_debug_bgp` / `show debugging bgp` / `bgp_config_write_debug()` arms (copy the `unreachability` lines at ~2375, ~2482, ~2627).

### 3.14 `vtysh/vtysh.c` — navigation

Mirror commit `4154e91508` (~50 lines). Add to `vtysh`:
- `DEFUNSH(VTYSH_BGPD, address_family_ipv4_crypto_routes, …)` style entries — or, since it uses the shared grammar, add the two nodes to the node table and the `bgp_afi_safi` navigation switch.
- `vtysh.c` has a big `switch (vty->node)` in `vtysh_exit()` / `node_parent()` — add `BGP_IPV4_CRYPTO_NODE` / `BGP_IPV6_CRYPTO_NODE` returning `BGP_NODE`.
- Register in `vtysh_init_cmd` / the generated `ctx` list.
- `extract.pl` picks up the new `install_element` calls automatically at build time.

### 3.15 `bgpd/subdir.am`

**No change in v1** (no new `.c`). If a `bgpd/bgp_crypto_routes.c` helper is later added (v2, for a custom attribute), add it there next to `bgp_unreach.c`:
```
+	bgpd/bgp_crypto_routes.c \
...
+	bgpd/bgp_crypto_routes.h \
```

### 3.16 `doc/user/` — user documentation

- `doc/user/crypto-routes.rst` — new page: overview, semantics (control-plane only, no FIB), config examples, `show`/`debug` reference, IANA-SAFI-241 caveat.
- `doc/user/bgp.rst` — add `crypto-routes` to the address-family list + a `.. toctree::` / `:ref:` link.

---

## 4. What Is Explicitly NOT Changed (and why that's correct)

| Not touched | Why safe |
|---|---|
| `zebra/`, ZAPI, `lib/zclient.c` per-value logic | v1 never sends crypto_routes to zebra. |
| `bgp_zebra.c` FIB gates | Deliberately excluded → no FIB install. (Only add trivial no-op `switch` arms if `-Wswitch` complains — the `SAFI_UNREACH` `bgp_zebra.c` change was GR-sync-specific, not needed here.) |
| `bgp_nht.c` NHT registration | crypto_routes nexthop is informational; no NHT in v1. Add no-op `switch` arm only. |
| `FOREACH_AFI_SAFI_NSF` | No GR/NSF participation in v1. |
| `struct bgp_path_info_extra` | No per-path extra state (contrast `unreach`). |
| `mgmtd`, `yang/` | No northbound in v1; config is DEFUN-based like most of bgpd. |
| Multipath code | `maxpaths` pinned to 1. |

---

## 5. Build & Codegen Notes

- **`clippy`**: `DEFPY` commands (debug, show) are preprocessed at build — run `./bootstrap.sh && ./configure` first so `lib/clippy` exists; `make` regenerates `*_clippy.c`. No manual step.
- **`-Wswitch` / `-Werror`**: the build **will fail** until every `switch (safi)` over `enum safi_t` without `default:` has the new arm. This is the feature working as intended — treat each failure as a checklist item. Enumerate up front:
  ```
  grep -rn "case SAFI_UNREACH:" bgpd/ lib/     # ~8 files → each needs case SAFI_CRYPTO_ROUTES:
  grep -rn "switch (safi)\|switch (packet->safi)\|switch (table->safi)" bgpd/ lib/
  ```
- **`vtysh`**: `vtysh/extract.pl` + `vtysh/vtysh_cmd.c` regenerate from the daemons' `install_element` calls on every build.
- **No new dependency, no `configure.ac` change, no new `--enable-*` flag.** (Optionally gate behind `--enable-bgp-crypto-routes` if the team wants it experimental — not recommended; adds ifdef noise.)

---

## 6. Reference Commit Breakdown (implementation order)

| # | Commit (`bgpd:`/`lib:` prefix, ≤50-char subject) | Files | ~LoC | Buildable? |
|---|---|---|---|---|
| 1 | `lib: add SAFI_CRYPTO_ROUTES to AFI/SAFI definitions` | `lib/zebra.h`, `lib/iana_afi.h`, `lib/prefix.c`, `bgpd/rfapi/rfapi_import.c`, `bgpd/rfapi/rfapi_monitor.c` | ~25 | yes (rfapi arms included) |
| 2 | `bgpd: handle crypto-routes SAFI in attr/packet processing` | `bgp_attr.c`, `bgp_packet.c` | ~25 | yes |
| 3 | `bgpd: handle crypto-routes SAFI in capability display` | `bgp_open.c` | ~15 | yes |
| 4 | `bgpd: add crypto-routes SAFI route processing + updgrp` | `bgp_route.c`, `bgp_route.h`, `bgp_updgrp_packet.c`, `bgp_bmp.c`, `bgp_nht.c`, `bgp_fsm.c`, misc | ~60 | yes |
| 5 | `bgpd: crypto-routes af_index, RIB, maxpaths` | `bgpd.h`, `bgpd.c` | ~45 | yes |
| 6 | `bgpd: add address-family crypto-routes config commands` | `lib/command.h`, `bgp_vty.c`, `bgp_vty.h`, `bgpd.c`, `vtysh/vtysh.c` | ~210 | yes (family activatable) |
| 7 | `bgpd: add 'show bgp [ipv4|ipv6] crypto-routes' command` | `bgp_vty.c`, `bgp_route.c` | ~40 | yes |
| 8 | `bgpd: add 'debug bgp crypto-routes' CLI` | `bgp_debug.c`, `bgp_debug.h` | ~40 | yes |
| 9 | `doc: add BGP crypto-routes SAFI user documentation` | `doc/user/crypto-routes.rst`, `doc/user/bgp.rst` | ~150 | n/a |
| 10 | `tests: add crypto-routes SAFI unit + topotest` | `tests/bgpd/test_mp_attr.c` (or new), `tests/topotests/bgp_crypto_routes/*` | ~400 | n/a |

Each commit builds and passes `make check` (bisectable — FRR workflow requirement). Every commit needs `Signed-off-by: <real name> <email>`.

---

## 7. Pre-Compilation Review Checklist

Reviewer verifies, per hunk, **without building**:

- [ ] `lib/zebra.h`: `SAFI_CRYPTO_ROUTES = 10`, `SAFI_MAX = 11`; **not** added to `FOREACH_AFI_SAFI_NSF`.
- [ ] `lib/iana_afi.h`: value `241` is in Private-Use (241–254) ✔; both `safi_iana2int` and `safi_int2iana` arms present and symmetric.
- [ ] `lib/prefix.c`: `safi2str` returns `"crypto-routes"` (hyphen, matches CLI token).
- [ ] `bgpd.h`: 2 `enum bgp_af_index` values added **before** `BGP_AF_MAX`; `afindex()` arms for `AFI_IP`, `AFI_IP6` return the right value; `AFI_L2VPN`/`AFI_BGP_LS` return `BGP_AF_MAX`.
- [ ] `bgpd.c`: `bgp_create()` maxpaths pin includes `SAFI_CRYPTO_ROUTES`; no accidental change to the `FOREACH_AFI_SAFI` alloc loop.
- [ ] `bgp_packet.c: bgp_nlri_parse()`: `SAFI_CRYPTO_ROUTES` shares the `bgp_nlri_parse_ip()` arm with `SAFI_UNICAST`.
- [ ] `bgp_route.c`: zebra-announce gate in `bgp_process_main_one()` **NOT** widened (grep `safi == SAFI_UNICAST` near `bgp_zebra_announce_actual` calls — must be unchanged); `subgroup_announce_check()` **NOT** modified (it is already SAFI-generic — verify no new arm was added there).
- [ ] All `case SAFI_UNREACH:` sites (`grep -rn "case SAFI_UNREACH:" bgpd/ lib/`) have a sibling `case SAFI_CRYPTO_ROUTES:` with equivalent (usually no-op) behaviour.
- [ ] `bgp_vty.c`: `bgp_node_type`, `bgp_node_afi`, `bgp_node_safi`, `bgp_vty_safi_from_str`, config-write frame + call, `cmd_node` structs, `install_node`/`install_default`, `install_element` block — all mirror the `BGP_IPV4U_NODE` pattern.
- [ ] `lib/command.h`: new nodes contiguous with other `BGP_*_NODE`s.
- [ ] `vtysh/vtysh.c`: `node_parent()` / exit switch returns `BGP_NODE` for both new nodes.
- [ ] Non-default BGP instance: crypto-routes rejected in `address_family_ipv4_safi_cmd` guard (core-instance only) — matches flowspec/labeled-unicast policy.
- [ ] Every new file has SPDX `GPL-2.0-or-later` + copyright; first include `<zebra.h>`; no `strcpy`/`sprintf`; typesafe containers if any new list.
- [ ] Doc page added; `bgp.rst` links it.
- [ ] Commits bisectable, imperative subjects, `Signed-off-by` present.

---

## Appendix A — Full `switch (safi)` / `case SAFI_*` site inventory (v1)

From `grep -rn "case SAFI_UNREACH:" bgpd/ lib/` on `bc8971168b` — each gets a `case SAFI_CRYPTO_ROUTES:` sibling:

```
lib/prefix.c                  safi2str()                         → "crypto-routes"
lib/iana_afi.h                safi_iana2int(), safi_int2iana()    → SAFI_CRYPTO_ROUTES / 241
bgpd/bgpd.h                   afindex() ×4, container helper      → BGP_AF_*_CRYPTO_ROUTES / BGP_AF_MAX
bgpd/bgp_attr.c               MP nexthop/NLRI guards             → treat as unicast
bgpd/bgp_packet.c             bgp_nlri_parse()                   → bgp_nlri_parse_ip
bgpd/bgp_open.c               capability name dump               → "crypto-routes"
bgpd/bgp_route.c              policy/show/announce switches      → treat as unicast; NOT FIB gate
bgpd/bgp_vty.c                node/str/config helpers            → BGP_*_CRYPTO_NODE
bgpd/rfapi/rfapi_import.c     fall-through group                 → no-op
bgpd/rfapi/rfapi_monitor.c    fall-through group                 → no-op
```
Plus `-Wswitch` may flag (add no-op arm): `bgp_bmp.c`, `bgp_nht.c`, `bgp_fsm.c`, `bgp_label.c`, `bgp_mplsvpn.c`, `bgp_rpki.c`, `bgp_mac.c`, `bgp_ls.c`, `bgp_evpn.c`, `bgp_evpn_mh.c`, `bgp_flowspec.c`, `bgp_updgrp_packet.c`. **Run the enumeration grep before starting.**

---

## Appendix B — If a New AFI Is Truly Required

Only if `crypto_routes` NLRI is **not** an IP prefix (e.g. a key-id + endpoint tuple). Then, in addition to everything above:

- `lib/zebra.h`: `AFI_CRYPTO = 5`, `AFI_MAX = 6` — **resizes `[AFI_MAX]` arrays across all of `lib/` and every daemon** (`if.c`, `rib.h`, `zclient`, `nexthop`, prefix machinery, OSPF/ISIS/… only compile-touched but must be checked).
- `lib/iana_afi.h`: `IANA_AFI_*` + `afi_iana2int`/`afi_int2iana`.
- `lib/prefix.h`: possibly a new `struct prefix_crypto` + `family` constant + `prefix_afi()` / `afi2family()` / `family2afi()`.
- `bgpd`: a real `bgp_nlri_parse_crypto()` + encoder, a `struct bgp_path_info_extra_crypto`, `bgp_crypto_routes.c/.h`, `subdir.am`.
- Effort: closer to the **full `bgp_unreach.c`** scale (~2,500 LoC) than the v1 estimate.

**Recommendation:** stay with SAFI-under-AFI_IP/IP6 unless a hard requirement forces otherwise.
