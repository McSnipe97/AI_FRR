# Code Changes — Implementing `crypto_routes` SAFI in FRRouting

**Status:** Implementation spec, ready to apply · **Date:** 2026-09-10
**Target:** FRR `master`, submodule `bc8971168b` (`frr-10.8.0-dev-1194-gbc8971168b`)
**Reads with:** [`HLD_v1.md`](./HLD_v1.md) · [`LLD_v1.md`](./LLD_v1.md) · validated by [`VALIDATION_PLAN.md`](./VALIDATION_PLAN.md)

This is the definitive, apply-ready list of every code change. Hunks are written against the verified current source. Line numbers are indicative; the anchor is always **"the arm next to `case SAFI_UNREACH:`"** (the merged `SAFI_UNREACH` series is the structural template — `git log --grep UNREACH`).

Legenda: ➕ add · ✏️ modify · 🆕 new file · 🚫 deliberately unchanged.

---

## 0. Scope Recap

- New **SAFI** `SAFI_CRYPTO_ROUTES` under existing `AFI_IP` / `AFI_IP6`. No new AFI.
- NLRI = plain IP prefix, wire-identical to unicast → reuse `bgp_nlri_parse_ip()` and the unicast prefix encoder.
- **Control-plane only**: not installed into zebra/FIB. No NHT, no GR-NSF, no ZAPI change.
- IANA SAFI `241` (Private-Use).
- No new source file. No `subdir.am` change. No `configure.ac` change. No new dependency.
- `bgp_update()` / `bgp_nlri_parse_ip()` signatures are **unchanged** (crypto_routes carries no out-of-band NLRI data — unlike `SAFI_UNREACH`).

---

## 1. Change Inventory (18 files)

| # | File | Kind | Purpose |
|---|---|---|---|
| C1.1 | `lib/zebra.h` | ✏️ | `SAFI_CRYPTO_ROUTES = 10`, `SAFI_MAX = 11` |
| C1.2 | `lib/iana_afi.h` | ✏️ | `IANA_SAFI_CRYPTO_ROUTES = 241` + 2 mapping arms |
| C1.3 | `lib/prefix.c` | ✏️ | `safi2str()` → `"crypto-routes"` |
| C1.4 | `lib/command.h` | ✏️ | `BGP_IPV4_CRYPTO_NODE`, `BGP_IPV6_CRYPTO_NODE` |
| C2.1 | `bgpd/bgpd.h` | ✏️ | `enum bgp_af_index` +2; `afindex()` +4 arms; `peer_afi_active_nego()`, `peer_group_af_configured()` |
| C2.2 | `bgpd/bgpd.c` | ✏️ | `bgp_create()` maxpaths pin |
| C3.1 | `bgpd/bgp_open.c` | ✏️ | `bgp_capability_vty_out()` +2 arms (capability display) |
| C3.2 | `bgpd/bgp_attr.c` | ✏️ | 4 `switch (safi)` arms (nexthop encode ×2, prefix encode, prefix size); `nh_afi` selection |
| C3.3 | `bgpd/bgp_packet.c` | ✏️ | `bgp_nlri_parse()` dispatch → `bgp_nlri_parse_ip()` |
| C4.1 | `bgpd/bgp_route.c` | ✏️ | `bgp_rd_from_dest()` arm; (optional) ext-comm JSON in `route_vty_out()` |
| C4.2 | `bgpd/rfapi/rfapi_import.c` | ✏️ | 3 no-op `switch (safi)` arms (VNC build) |
| C4.3 | `bgpd/rfapi/rfapi_monitor.c` | ✏️ | 2 no-op `switch (safi)` arms (VNC build) |
| C5.1 | `bgpd/bgp_vty.h` | ✏️ | `BGP_SAFI_WITH_LABEL_CMD_STR` / `..._HELP_STR` grammar |
| C5.2 | `bgpd/bgp_vty.c` | ✏️ | nodes, grammar, `bgp_node_*`, `bgp_vty_safi_from_str`, `argv_find_and_parse_safi`, config-write, `install_element` block, `cmd_node` structs |
| C5.3 | `vtysh/vtysh.c` | ✏️ | 2 nodes + navigation DEFUNSH + registration |
| C6.1 | `bgpd/bgp_vty.c` / `bgp_route.c` | ✏️ | `show bgp [ipv4\|ipv6] crypto-routes …` grammar tokens |
| C7.1 | `bgpd/bgp_debug.c` + `bgpd/bgp_debug.h` | ✏️ | `debug bgp crypto-routes` |
| C8.1 | `doc/user/crypto-routes.rst` + `doc/user/bgp.rst` + `doc/user/index.rst` | 🆕/✏️ | user documentation |
| C9.1 | `tests/bgpd/test_mp_attr.c` / `test_capability.c` | ✏️ | unit assertions |
| C9.2 | `tests/topotests/bgp_crypto_routes/` | 🆕 | integration test |

---

## 2. Commit 1 — `lib: add SAFI_CRYPTO_ROUTES to AFI/SAFI definitions`

### C1.1 `lib/zebra.h`

```c
 typedef enum {
 	SAFI_UNSPEC = 0,
 	SAFI_UNICAST = 1,
 	...
 	SAFI_BGP_LS = 8, /* BGP-LS (RFC 9552) */
 	SAFI_UNREACH = 9,
-	SAFI_MAX = 10
+	SAFI_CRYPTO_ROUTES = 10, /* crypto/overlay-reachable prefixes; control-plane only */
+	SAFI_MAX = 11
 } safi_t;
```

🚫 **Do NOT** add `SAFI_CRYPTO_ROUTES` to `FOREACH_AFI_SAFI_NSF` — v1 does not participate in Graceful-Restart NSF.

### C1.2 `lib/iana_afi.h`

```c
 typedef enum {
 	IANA_SAFI_RESERVED = 0,
 	...
 	IANA_SAFI_MPLS_VPN = 128,
-	IANA_SAFI_FLOWSPEC = 133 /* Flowspec per RFC 8955 */
+	IANA_SAFI_FLOWSPEC = 133, /* Flowspec per RFC 8955 */
+	IANA_SAFI_CRYPTO_ROUTES = 241 /* IANA "Reserved for Private Use" (241-254) */
 } iana_safi_t;
```

```c
 static inline safi_t safi_iana2int(iana_safi_t safi)
 {
 	switch (safi) {
 	...
 	case IANA_SAFI_FLOWSPEC:
 		return SAFI_FLOWSPEC;
+	case IANA_SAFI_CRYPTO_ROUTES:
+		return SAFI_CRYPTO_ROUTES;
 	case IANA_SAFI_BGP_LS:
 		return SAFI_BGP_LS;
 	case IANA_SAFI_RESERVED:
 		return SAFI_MAX;
 	}
 	return SAFI_MAX;
 }

 static inline iana_safi_t safi_int2iana(safi_t safi)
 {
 	switch (safi) {
 	...
 	case SAFI_FLOWSPEC:
 		return IANA_SAFI_FLOWSPEC;
+	case SAFI_CRYPTO_ROUTES:
+		return IANA_SAFI_CRYPTO_ROUTES;
 	case SAFI_BGP_LS:
 		return IANA_SAFI_BGP_LS;
 	case SAFI_UNSPEC:
 	case SAFI_MAX:
 		return IANA_SAFI_RESERVED;
 	}
 	return IANA_SAFI_RESERVED;
 }
```

> After this hunk, `bgp_map_afi_safi_iana2int()` / `..._int2iana()` (`bgpd/bgpd.c:1000`) work for the new SAFI with no further change — and MP-capability advertise/receive + MP_REACH/MP_UNREACH AFI/SAFI framing become functional.

### C1.3 `lib/prefix.c`

```c
 const char *safi2str(safi_t safi)
 {
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
 	assert(!"Reached end of function we should never reach");
 	return "DEV ESCAPE";
 }
```

### C1.4 `lib/command.h`

```c
 enum node_type {
 	...
 	BGP_FLOWSPECV4_NODE,	/* BGP IPv4 FLOWSPEC Address-Family */
 	BGP_FLOWSPECV6_NODE,	/* BGP IPv6 FLOWSPEC Address-Family */
 	BGP_IPV4U_NODE,		/* BGP IPv4 unreachability address family. */
 	BGP_IPV6U_NODE,		/* BGP IPv6 unreachability address family. */
+	BGP_IPV4_CRYPTO_NODE,	/* BGP IPv4 crypto-routes address family. */
+	BGP_IPV6_CRYPTO_NODE,	/* BGP IPv6 crypto-routes address family. */
 	BFD_NODE,		 /* BFD protocol mode. */
 	...
 };
```

**rfapi arms (same commit, VNC build must compile):**

### C4.2 `bgpd/rfapi/rfapi_import.c` — 3 sites (~205, ~3823, ~4068)

```c
 	case SAFI_BGP_LS:
+	case SAFI_CRYPTO_ROUTES:
 	case SAFI_UNREACH:
 	case SAFI_UNSPEC:
 	case SAFI_UNICAST:
 	case SAFI_MULTICAST:
 	...
```

### C4.3 `bgpd/rfapi/rfapi_monitor.c` — 2 sites (~188, ~265)

Same one-line `case SAFI_CRYPTO_ROUTES:` added to the `case SAFI_BGP_LS: … case SAFI_UNREACH:` fall-through group.

**Commit builds** with `--enable-bgp-vnc` and without.

---

## 3. Commit 2 — `bgpd: add crypto-routes af_index and RIB sizing`

### C2.1 `bgpd/bgpd.h`

**`enum bgp_af_index`:**
```c
 enum bgp_af_index {
 	...
 	BGP_AF_BGP_LS,
 	BGP_AF_IPV4_UNREACH,
 	BGP_AF_IPV6_UNREACH,
+	BGP_AF_IPV4_CRYPTO_ROUTES,
+	BGP_AF_IPV6_CRYPTO_ROUTES,
 	BGP_AF_MAX
 };
```

**`afindex()` — 4 arms:**
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
 	case AFI_IP6:
 		switch (safi) {
 		...
 		case SAFI_UNREACH:
 			return BGP_AF_IPV6_UNREACH;
+		case SAFI_CRYPTO_ROUTES:
+			return BGP_AF_IPV6_CRYPTO_ROUTES;
 		...
 		}
 	case AFI_L2VPN:  /* add `case SAFI_CRYPTO_ROUTES:` to the `return BGP_AF_MAX` group */
 	case AFI_BGP_LS: /* add `case SAFI_CRYPTO_ROUTES:` to the `return BGP_AF_MAX` group */
```

**`peer_afi_active_nego()` — add to the OR chain:**
```c
 	if (peer->afc_nego[afi][SAFI_UNICAST] || peer->afc_nego[afi][SAFI_MULTICAST] ||
 	    peer->afc_nego[afi][SAFI_LABELED_UNICAST] || peer->afc_nego[afi][SAFI_MPLS_VPN] ||
 	    peer->afc_nego[afi][SAFI_ENCAP] || peer->afc_nego[afi][SAFI_FLOWSPEC] ||
-	    peer->afc_nego[afi][SAFI_UNREACH] || peer->afc_nego[afi][SAFI_EVPN])
+	    peer->afc_nego[afi][SAFI_UNREACH] || peer->afc_nego[afi][SAFI_CRYPTO_ROUTES] ||
+	    peer->afc_nego[afi][SAFI_EVPN])
 		return 1;
```

**`peer_group_af_configured()` — add `SAFI_CRYPTO_ROUTES` for `AFI_IP` and `AFI_IP6`:**
```c
 	    || peer->afc[AFI_IP][SAFI_UNREACH]
+	    || peer->afc[AFI_IP][SAFI_CRYPTO_ROUTES]
 	    || peer->afc[AFI_IP6][SAFI_UNICAST]
 	...
 	    || peer->afc[AFI_IP6][SAFI_UNREACH]
+	    || peer->afc[AFI_IP6][SAFI_CRYPTO_ROUTES]
 	    || peer->afc[AFI_L2VPN][SAFI_EVPN]
```

### C2.2 `bgpd/bgpd.c` — `bgp_create()` (~4056)

```c
 		/* Enable maximum-paths */
-		/* For SAFI_UNREACH, hardcode max-paths to 1 (only one best path) */
-		if (safi == SAFI_UNREACH) {
+		/*
+		 * SAFI_UNREACH and SAFI_CRYPTO_ROUTES are informational
+		 * control-plane RIBs; pin max-paths to 1 (single authoritative
+		 * path, no ECMP semantics).
+		 */
+		if (safi == SAFI_UNREACH || safi == SAFI_CRYPTO_ROUTES) {
 			bgp_maximum_paths_set(bgp, afi, safi, BGP_PEER_EBGP, 1, 0);
 			bgp_maximum_paths_set(bgp, afi, safi, BGP_PEER_IBGP, 1, 0);
 		} else {
 			bgp_maximum_paths_set(bgp, afi, safi, BGP_PEER_EBGP, multipath_num, 0);
 			bgp_maximum_paths_set(bgp, afi, safi, BGP_PEER_IBGP, multipath_num, 0);
 		}
```

🚫 The `FOREACH_AFI_SAFI` loop already calls `bgp_table_init()` for `rib`/`static_routes`/`aggregate` → `bgp->rib[AFI_IP][SAFI_CRYPTO_ROUTES]` is allocated **with no change**. `bgp_free()`'s `FOREACH_AFI_SAFI` frees it likewise.

---

## 4. Commit 3 — `bgpd: handle crypto-routes SAFI in capability/attr/packet`

### C3.1 `bgpd/bgp_open.c` — `bgp_capability_vty_out()` (2 arms, ~222 and ~282)

```c
 				case SAFI_UNREACH:
 					json_object_string_add(json_cap,
 							       "capabilityErrorMultiProtocolSafi",
 							       "unreachability");
 					break;
+				case SAFI_CRYPTO_ROUTES:
+					json_object_string_add(json_cap,
+							       "capabilityErrorMultiProtocolSafi",
+							       "crypto-routes");
+					break;
 				case SAFI_UNSPEC:
 				case SAFI_MAX:
```
```c
 				case SAFI_UNREACH:
 					vty_out(vty, "SAFI Unreachability");
 					break;
+				case SAFI_CRYPTO_ROUTES:
+					vty_out(vty, "SAFI Crypto-Routes");
+					break;
 				case SAFI_UNSPEC:
 				case SAFI_MAX:
```

🚫 `bgp_capability_mp()` (receive) and `bgp_open_capability()` (advertise) are generic `FOREACH_AFI_SAFI` loops keyed on `peer->afc[afi][safi]` + the IANA mapping — **no change**.

### C3.2 `bgpd/bgp_attr.c`

**(a) `nh_afi` selection in `bgp_packet_mpattr_start()` (~4928)** — give crypto-routes the same enhanced-nexthop treatment as unicast:
```c
-	if ((safi == SAFI_UNICAST || safi == SAFI_LABELED_UNICAST
-	     || safi == SAFI_MPLS_VPN || safi == SAFI_MULTICAST))
+	if ((safi == SAFI_UNICAST || safi == SAFI_LABELED_UNICAST
+	     || safi == SAFI_MPLS_VPN || safi == SAFI_MULTICAST
+	     || safi == SAFI_CRYPTO_ROUTES))
 		nh_afi = peer_cap_enhe(peer, afi, safi) ? AFI_IP6 : AFI_IP;
```

**(b) AFI_IP nexthop `switch (safi)` (~4945)** — join the unicast arm:
```c
 		switch (safi) {
 		case SAFI_UNICAST:
 		case SAFI_MULTICAST:
 		case SAFI_LABELED_UNICAST:
+		case SAFI_CRYPTO_ROUTES:
 			stream_putc(s, 4);
 			stream_put_ipv4(s, attr->nexthop.s_addr);
 			break;
```

**(c) AFI_IP6 nexthop `switch (safi)` (~4995)** — join the unicast arm:
```c
 		switch (safi) {
 		case SAFI_UNICAST:
 		case SAFI_MULTICAST:
 		case SAFI_LABELED_UNICAST:
 		case SAFI_EVPN: {
+		case SAFI_CRYPTO_ROUTES:
 			if (attr->mp_nexthop_len == BGP_ATTR_NHLEN_IPV6_GLOBAL_AND_LL) {
 			...
 		} break;
```
> Note: `SAFI_EVPN` opens a `{` block here. Put `case SAFI_CRYPTO_ROUTES:` **before** the `{` so it shares the block, matching the other three labels.

**(d) `bgp_packet_mpattr_prefix()` `switch (safi)` (~5208)** — join unicast/multicast:
```c
 	case SAFI_UNICAST:
 	case SAFI_MULTICAST:
+	case SAFI_CRYPTO_ROUTES:
 		bgp_attr_stream_put_prefix_addpath(s, p, addpath_capable, addpath_tx_id);
 		break;
```

**(e) `bgp_packet_mpattr_prefix_size()` `switch (safi)` (~5333)** — join unicast/multicast (no size delta):
```c
 	case SAFI_UNICAST:
 	case SAFI_MULTICAST:
+	case SAFI_CRYPTO_ROUTES:
 		break;
```

🚫 `bgp_mp_reach_parse()` nexthop-length `case 0:` guard stays as-is — a zero-length nexthop for crypto-routes is correctly an error.

### C3.3 `bgpd/bgp_packet.c` — `bgp_nlri_parse()` (~317)

```c
 	switch (packet->safi) {
 	case SAFI_UNICAST:
 	case SAFI_MULTICAST:
+	case SAFI_CRYPTO_ROUTES:
 		return bgp_nlri_parse_ip(peer, mp_withdraw ? NULL : attr,
 					 packet);
```

> This is the **entire receive-path NLRI implementation.** `bgp_nlri_parse_ip()` already calls `bgp_update()/bgp_withdraw()` with `(afi, safi)` from the packet and passes `NULL` for the trailing `unreach` params — no signature change.

---

## 5. Commit 4 — `bgpd: crypto-routes SAFI route processing`

### C4.1 `bgpd/bgp_route.c`

**`bgp_rd_from_dest()` (~16029)** — crypto-routes has no RD:
```c
 	case SAFI_BGP_LS:
 	case SAFI_UNSPEC:
 	case SAFI_UNICAST:
 	case SAFI_MULTICAST:
 	case SAFI_LABELED_UNICAST:
 	case SAFI_FLOWSPEC:
 	case SAFI_UNREACH:
+	case SAFI_CRYPTO_ROUTES:
 	case SAFI_MAX:
 		return NULL;
```

**(optional, parity) `route_vty_out()` ext-community JSON (~11848):**
```c
-		if ((safi == SAFI_EVPN || safi == SAFI_UNREACH) &&
+		if ((safi == SAFI_EVPN || safi == SAFI_UNREACH ||
+		     safi == SAFI_CRYPTO_ROUTES) &&
 		    bgp_attr_exists(attr, BGP_ATTR_EXT_COMMUNITIES)) {
```

🚫 **Zebra announce gate** in `bgp_process_main_one()` (`(afi == AFI_IP || afi == AFI_IP6) && safi == SAFI_UNICAST`) — **NOT widened**. crypto-routes falls through → never installed to FIB.
🚫 `subgroup_announce_check()` — SAFI-generic, **no change**.
🚫 `bgp_update_martian_nexthop()` — already returns valid for non-unicast/multicast/EVPN, **no change**.
🚫 `bgp_path_info_extra_free()` — no `e->crypto` member; **no change** (contrast `e->unreach`).

### Local origination — `network` command

The `network`/`no network` commands (`bgp_static_set()` / `bgp_static_unset()`) are SAFI-generic via `bgp_node_safi(vty)`. Wiring is in Commit 5 (`install_element` on the new nodes).

---

## 6. Commit 5 — `bgpd: add address-family crypto-routes configuration commands`

### C5.1 `bgpd/bgp_vty.h`

```c
 #define BGP_SAFI_WITH_LABEL_CMD_STR                                            \
-	"<unicast|multicast|vpn|labeled-unicast|flowspec|unreachability>"
+	"<unicast|multicast|vpn|labeled-unicast|flowspec|unreachability|crypto-routes>"
 #define BGP_SAFI_WITH_LABEL_HELP_STR                                           \
 	BGP_AF_MODIFIER_STR BGP_AF_MODIFIER_STR BGP_AF_MODIFIER_STR             \
-		BGP_AF_MODIFIER_STR BGP_AF_MODIFIER_STR BGP_AF_MODIFIER_STR
+		BGP_AF_MODIFIER_STR BGP_AF_MODIFIER_STR BGP_AF_MODIFIER_STR    \
+		BGP_AF_MODIFIER_STR
```
(7 `BGP_AF_MODIFIER_STR` = 7 tokens in the `<...>` alternation.)

### C5.2 `bgpd/bgp_vty.c`

**`bgp_node_type()` — 2 arms (~186, ~209):**
```c
 		case SAFI_UNREACH:
 			return BGP_IPV4U_NODE;
+		case SAFI_CRYPTO_ROUTES:
+			return BGP_IPV4_CRYPTO_NODE;
```
```c
 		case SAFI_UNREACH:
 			return BGP_IPV6U_NODE;
+		case SAFI_CRYPTO_ROUTES:
+			return BGP_IPV6_CRYPTO_NODE;
```
> Also add `case SAFI_CRYPTO_ROUTES:` to the "not expected" fall-through groups for `AFI_IP`/`AFI_IP6` if the switch is exhaustive (it lists every SAFI) — check and match `SAFI_BGP_LS` placement.

**`get_afi_safi_vty_str()` (~249, ~264):**
```c
 		if (safi == SAFI_UNREACH)
 			return "IPv4 Unreachability";
+		if (safi == SAFI_CRYPTO_ROUTES)
+			return "IPv4 Crypto-Routes";
```
(+ `"IPv6 Crypto-Routes"` in the AFI_IP6 branch)

**`get_afi_safi_json_str()` (~298, ~313):**
```c
 		if (safi == SAFI_UNREACH)
 			return "ipv4Unreachability";
+		if (safi == SAFI_CRYPTO_ROUTES)
+			return "ipv4CryptoRoutes";
```
(+ `"ipv6CryptoRoutes"`)

**`bgp_node_afi()` (~469):**
```c
 	case BGP_FLOWSPECV6_NODE:
 	case BGP_IPV6U_NODE:
+	case BGP_IPV6_CRYPTO_NODE:
 		afi = AFI_IP6;
 		break;
```

**`bgp_node_safi()` (~513):**
```c
 	case BGP_IPV4U_NODE:
 	case BGP_IPV6U_NODE:
 		safi = SAFI_UNREACH;
 		break;
+	case BGP_IPV4_CRYPTO_NODE:
+	case BGP_IPV6_CRYPTO_NODE:
+		safi = SAFI_CRYPTO_ROUTES;
+		break;
```

**`bgp_vty_safi_from_str()` (~581):**
```c
 	else if (strmatch(safi_str, "unreachability"))
 		safi = SAFI_UNREACH;
+	else if (strmatch(safi_str, "crypto-routes"))
+		safi = SAFI_CRYPTO_ROUTES;
 	return safi;
```

**`argv_find_and_parse_safi()` (~614):**
```c
 	} else if (argv_find(argv, argc, "unreachability", index)) {
 		ret = 1;
 		if (safi)
 			*safi = SAFI_UNREACH;
+	} else if (argv_find(argv, argc, "crypto-routes", index)) {
+		ret = 1;
+		if (safi)
+			*safi = SAFI_CRYPTO_ROUTES;
 	}
```

**`get_bgp_default_af_flag()` — 4 arms (~658, ~682, ~700, ~717):**
```c
 		case SAFI_UNREACH:
 			return "ipv4-unreachability";
+		case SAFI_CRYPTO_ROUTES:
+			return "ipv4-crypto-routes";
```
(+ `"ipv6-crypto-routes"`; + `case SAFI_CRYPTO_ROUTES:` to the two `"unknown-afi/safi"` groups for L2VPN and BGP_LS)

**`bgp_maxpaths_config_vty()` (~2360) — reject max-paths change:**
```c
-	if (safi == SAFI_UNREACH) {
+	if (safi == SAFI_UNREACH || safi == SAFI_CRYPTO_ROUTES) {
 		vty_out(vty,
-			"%% maximum-paths is fixed at 1 for unreachability\n");
+			"%% maximum-paths is fixed at 1 for %s\n", safi2str(safi));
 		return CMD_WARNING_CONFIG_FAILED;
 	}
```

**`bgp_config_write_maxpaths()` (~2999) — suppress stray line:**
```c
-	if (safi == SAFI_UNREACH)
+	if (safi == SAFI_UNREACH || safi == SAFI_CRYPTO_ROUTES)
 		return;
```

**`address_family_ipv4_safi_cmd` / `address_family_ipv6_safi_cmd` grammar** — the DEFUN command string picks up the extra token from the widened `BGP_SAFI_WITH_LABEL_*` macro automatically. Confirm the non-default-instance guard stays as-is (crypto-routes not in `{UNICAST, MULTICAST, EVPN}` → rejected in vrf/view instances, matching flowspec).

**`exit_address_family` DEFUN (~12058):**
```c
 	    || vty->node == BGP_IPV4U_NODE
 	    || vty->node == BGP_IPV6U_NODE
+	    || vty->node == BGP_IPV4_CRYPTO_NODE
+	    || vty->node == BGP_IPV6_CRYPTO_NODE)
 		vty->node = BGP_NODE;
```

**`clear_ip_bgp_all_cmd` grammar (~12150)** — add `|crypto-routes` to the `[<unicast|…|unreachability>]` alternation string + one `"Address Family modifier\n"` help line.

**`bgp_config_write_family()` frame (~22040, ~22055):**
```c
 		else if (safi == SAFI_UNREACH)
 			vty_frame(vty, "ipv4 unreachability");
+		else if (safi == SAFI_CRYPTO_ROUTES)
+			vty_frame(vty, "ipv4 crypto-routes");
```
(+ `"ipv6 crypto-routes"`)

**`bgp_config_write()` (~22734, ~22756):**
```c
+		/* IPv4 Crypto-Routes configuration.  */
+		bgp_config_write_family(vty, bgp, AFI_IP, SAFI_CRYPTO_ROUTES);
```
(+ AFI_IP6, placed after the SAFI_UNREACH calls)

**`cmd_node` structs (~22882):**
```c
+static struct cmd_node bgp_ipv4_crypto_routes_node = {
+	.name = "bgp ipv4 crypto-routes",
+	.node = BGP_IPV4_CRYPTO_NODE,
+	.parent_node = BGP_NODE,
+	.prompt = "%s(config-router-af)# ",
+	.no_xpath = true,
+};
+
+static struct cmd_node bgp_ipv6_crypto_routes_node = {
+	.name = "bgp ipv6 crypto-routes",
+	.node = BGP_IPV6_CRYPTO_NODE,
+	.parent_node = BGP_NODE,
+	.prompt = "%s(config-router-af)# ",
+	.no_xpath = true,
+};
```

**`bgp_vty_init()`:**
```c
 	install_node(&bgp_ipv4_unreachability_node);
 	install_node(&bgp_ipv6_unreachability_node);
+	install_node(&bgp_ipv4_crypto_routes_node);
+	install_node(&bgp_ipv6_crypto_routes_node);
 	...
 	install_default(BGP_IPV4U_NODE);
 	install_default(BGP_IPV6U_NODE);
+	install_default(BGP_IPV4_CRYPTO_NODE);
+	install_default(BGP_IPV6_CRYPTO_NODE);
```

**`install_element` block** — for each of `BGP_IPV4_CRYPTO_NODE` and `BGP_IPV6_CRYPTO_NODE`, replicate every `install_element(BGP_IPV4U_NODE, …)` line (grep gives ~11 command families):
```c
	install_element(BGP_IPV4_CRYPTO_NODE, &neighbor_activate_cmd);
	install_element(BGP_IPV4_CRYPTO_NODE, &no_neighbor_activate_cmd);
	install_element(BGP_IPV4_CRYPTO_NODE, &neighbor_route_map_cmd);
	install_element(BGP_IPV4_CRYPTO_NODE, &no_neighbor_route_map_cmd);
	install_element(BGP_IPV4_CRYPTO_NODE, &neighbor_prefix_list_cmd);
	install_element(BGP_IPV4_CRYPTO_NODE, &no_neighbor_prefix_list_cmd);
	install_element(BGP_IPV4_CRYPTO_NODE, &neighbor_filter_list_cmd);
	install_element(BGP_IPV4_CRYPTO_NODE, &no_neighbor_filter_list_cmd);
	install_element(BGP_IPV4_CRYPTO_NODE, &neighbor_distribute_list_cmd);
	install_element(BGP_IPV4_CRYPTO_NODE, &no_neighbor_distribute_list_cmd);
	install_element(BGP_IPV4_CRYPTO_NODE, &neighbor_maximum_prefix_cmd);
	install_element(BGP_IPV4_CRYPTO_NODE, &neighbor_maximum_prefix_threshold_cmd);
	install_element(BGP_IPV4_CRYPTO_NODE, &neighbor_maximum_prefix_warning_cmd);
	install_element(BGP_IPV4_CRYPTO_NODE, &neighbor_maximum_prefix_threshold_warning_cmd);
	install_element(BGP_IPV4_CRYPTO_NODE, &neighbor_maximum_prefix_restart_cmd);
	install_element(BGP_IPV4_CRYPTO_NODE, &neighbor_maximum_prefix_threshold_restart_cmd);
	install_element(BGP_IPV4_CRYPTO_NODE, &no_neighbor_maximum_prefix_cmd);
	install_element(BGP_IPV4_CRYPTO_NODE, &neighbor_maximum_prefix_out_cmd);
	install_element(BGP_IPV4_CRYPTO_NODE, &no_neighbor_maximum_prefix_out_cmd);
	install_element(BGP_IPV4_CRYPTO_NODE, &neighbor_allowas_in_cmd);
	install_element(BGP_IPV4_CRYPTO_NODE, &no_neighbor_allowas_in_cmd);
	install_element(BGP_IPV4_CRYPTO_NODE, &neighbor_route_reflector_client_cmd);
	install_element(BGP_IPV4_CRYPTO_NODE, &no_neighbor_route_reflector_client_cmd);
	install_element(BGP_IPV4_CRYPTO_NODE, &neighbor_soft_reconfiguration_cmd);
	install_element(BGP_IPV4_CRYPTO_NODE, &no_neighbor_soft_reconfiguration_cmd);
	install_element(BGP_IPV4_CRYPTO_NODE, &bgp_network_cmd);        /* local origination */
	install_element(BGP_IPV4_CRYPTO_NODE, &no_bgp_network_cmd);
	install_element(BGP_IPV4_CRYPTO_NODE, &bgp_table_map_cmd);
	install_element(BGP_IPV4_CRYPTO_NODE, &no_bgp_table_map_cmd);
	install_element(BGP_IPV4_CRYPTO_NODE, &bgp_maxpaths_cmd);      /* returns the "fixed at 1" reject */
	install_element(BGP_IPV4_CRYPTO_NODE, &exit_address_family_cmd);
	/* ... identical block for BGP_IPV6_CRYPTO_NODE, with bgp_network_cmd → ipv6_bgp_network_cmd ... */
```

**`bgp_debug.c` node membership (~12120):**
```c
 	    || vty->node == BGP_IPV4U_NODE
 	    || vty->node == BGP_IPV6U_NODE
+	    || vty->node == BGP_IPV4_CRYPTO_NODE
+	    || vty->node == BGP_IPV6_CRYPTO_NODE)
```

### C5.3 `vtysh/vtysh.c`

```c
+static struct cmd_node bgp_ipv4_crypto_routes_node = {
+	.name = "bgp ipv4 crypto-routes",
+	.node = BGP_IPV4_CRYPTO_NODE,
+	.parent_node = BGP_NODE,
+	.prompt = "%s(config-router-af)# ",
+	.no_xpath = true,
+};
+static struct cmd_node bgp_ipv6_crypto_routes_node = { /* ...BGP_IPV6_CRYPTO_NODE... */ };
```
```c
+DEFUNSH(VTYSH_BGPD, address_family_ipv4_crypto_routes, address_family_ipv4_crypto_routes_cmd,
+	"address-family ipv4 crypto-routes",
+	"Enter Address Family command mode\n" BGP_AF_STR BGP_AF_MODIFIER_STR)
+{
+	vty->node = BGP_IPV4_CRYPTO_NODE;
+	return CMD_SUCCESS;
+}
+DEFUNSH(VTYSH_BGPD, address_family_ipv6_crypto_routes, address_family_ipv6_crypto_routes_cmd,
+	"address-family ipv6 crypto-routes",
+	"Enter Address Family command mode\n" BGP_AF_STR BGP_AF_MODIFIER_STR)
+{
+	vty->node = BGP_IPV6_CRYPTO_NODE;
+	return CMD_SUCCESS;
+}
```
```c
 DEFUNSH(VTYSH_BGPD, exit_address_family, exit_address_family_cmd, ...)
 	    || vty->node == BGP_IPV4U_NODE
 	    || vty->node == BGP_IPV6U_NODE
+	    || vty->node == BGP_IPV4_CRYPTO_NODE
+	    || vty->node == BGP_IPV6_CRYPTO_NODE)
 		vty->node = BGP_NODE;
```
```c
 void vtysh_init_vty(void)
 	...
 	install_node(&bgp_ipv4_unreachability_node);
 	install_node(&bgp_ipv6_unreachability_node);
+	install_node(&bgp_ipv4_crypto_routes_node);
+	install_node(&bgp_ipv6_crypto_routes_node);
 	...
+	install_element(BGP_NODE, &address_family_ipv4_crypto_routes_cmd);
+	install_element(BGP_IPV4_CRYPTO_NODE, &vtysh_exit_bgpd_cmd);
+	install_element(BGP_IPV4_CRYPTO_NODE, &vtysh_quit_bgpd_cmd);
+	install_element(BGP_IPV4_CRYPTO_NODE, &vtysh_end_all_cmd);
+	install_element(BGP_IPV4_CRYPTO_NODE, &exit_address_family_cmd);
+	install_element(BGP_NODE, &address_family_ipv6_crypto_routes_cmd);
+	install_element(BGP_IPV6_CRYPTO_NODE, &vtysh_exit_bgpd_cmd);
+	install_element(BGP_IPV6_CRYPTO_NODE, &vtysh_quit_bgpd_cmd);
+	install_element(BGP_IPV6_CRYPTO_NODE, &vtysh_end_all_cmd);
+	install_element(BGP_IPV6_CRYPTO_NODE, &exit_address_family_cmd);
```

> `vtysh/extract.pl` auto-harvests the new `install_element` calls in `bgpd/bgp_vty.c` at build time, so `neighbor`/`network` sub-commands work from `vtysh` with no further wiring.

---

## 7. Commit 6 — `bgpd: add 'show bgp [ipv4|ipv6] crypto-routes' command`

The generic `show bgp <afi> <safi> …` DEFPYs parse the SAFI token via `bgp_vty_safi_from_str()` / `argv_find_and_parse_safi()` (both extended in Commit 5). Add `crypto-routes` to the SAFI alternation in the relevant DEFPY command strings — grep `"unreachability"` in `bgpd/bgp_vty.c` and `bgpd/bgp_route.c` DEFPY/DEFUN strings (≈ lines 12214, 15298, 18977 and the `bgp_route.c` `show bgp` family):

```
"show [ip] bgp [<view|vrf> VIEWVRFNAME] [<ipv4|ipv6> [<unicast|multicast|...|unreachability|crypto-routes>]] ..."
```
plus one `BGP_AF_MODIFIER_STR` / help line per string.

`bgp_show()` / `bgp_show_table()` are generic table walkers — verify (they contain no `assert(safi != …)` excluding new SAFIs) and no code change needed. `show bgp summary` per-AF output works via `BGP_AF_IPV4_CRYPTO_ROUTES` + `get_afi_safi_vty_str()`.

---

## 8. Commit 7 — `bgpd: add 'debug bgp crypto-routes' CLI`

### `bgpd/bgp_debug.h`

```c
+extern unsigned long conf_bgp_debug_crypto_routes;
+extern unsigned long term_bgp_debug_crypto_routes;
 ...
+#define BGP_DEBUG_CRYPTO_ROUTES 0x01
```

### `bgpd/bgp_debug.c`

```c
+unsigned long conf_bgp_debug_crypto_routes;
+unsigned long term_bgp_debug_crypto_routes;
 ...
+DEFPY(debug_bgp_crypto_routes, debug_bgp_crypto_routes_cmd,
+      "[no$no] debug bgp crypto-routes",
+      NO_STR DEBUG_STR BGP_STR
+      "BGP crypto-routes (SAFI_CRYPTO_ROUTES) debugging\n")
+{
+	if (vty->node == CONFIG_NODE) {
+		if (no)
+			DEBUG_OFF(crypto_routes, CRYPTO_ROUTES);
+		else
+			DEBUG_ON(crypto_routes, CRYPTO_ROUTES);
+	} else {
+		if (no) {
+			TERM_DEBUG_OFF(crypto_routes, CRYPTO_ROUTES);
+			vty_out(vty, "BGP crypto-routes debugging is off\n");
+		} else {
+			TERM_DEBUG_ON(crypto_routes, CRYPTO_ROUTES);
+			vty_out(vty, "BGP crypto-routes debugging is on\n");
+		}
+	}
+	return CMD_SUCCESS;
+}
```
+ `bgp_debug_init()`: `install_element(ENABLE_NODE, &debug_bgp_crypto_routes_cmd); install_element(CONFIG_NODE, &debug_bgp_crypto_routes_cmd);`
+ `no_debug_bgp` / `bgp_debug_reset()`: `TERM_DEBUG_OFF(crypto_routes, CRYPTO_ROUTES);`
+ `show_debugging_bgp()`: `if (BGP_DEBUG(crypto_routes, CRYPTO_ROUTES)) vty_out(vty, "  BGP crypto-routes debugging is on\n");`
+ `bgp_config_write_debug()`: `if (CONF_BGP_DEBUG(crypto_routes, CRYPTO_ROUTES)) { vty_out(vty, "debug bgp crypto-routes\n"); write++; }`
+ Emit `zlog_debug()` under `BGP_DEBUG(crypto_routes, CRYPTO_ROUTES)` in `bgp_update()` / `bgp_withdraw()` / `subgroup_update_packet()` when `safi == SAFI_CRYPTO_ROUTES` (optional, small).

---

## 9. Commit 8 — `doc: BGP crypto-routes SAFI user documentation`

- 🆕 `doc/user/crypto-routes.rst` — overview (control-plane only, IANA SAFI 241 private-use caveat), config reference (`address-family ipv4|ipv6 crypto-routes`, `neighbor … activate`, `network`, route-maps), `show`/`debug` reference, worked example.
- ✏️ `doc/user/bgp.rst` — add `crypto-routes` to the address-family list; add `:ref:` link.
- ✏️ `doc/user/index.rst` (or `subdir.am` toctree) — add `crypto-routes` page.

Build check: `./configure --enable-doc && make -C doc html` (and `man`).

---

## 10. Commit 9 — `tests: crypto-routes SAFI unit + topotest`

### Unit (`tests/bgpd/`)

Extend `test_capability.c` and `test_mp_attr.c` (no new `subdir.am` entry needed if extending):

```c
/* test_capability.c */
assert(safi_int2iana(SAFI_CRYPTO_ROUTES) == IANA_SAFI_CRYPTO_ROUTES);      /* 241 */
assert(safi_iana2int(IANA_SAFI_CRYPTO_ROUTES) == SAFI_CRYPTO_ROUTES);
assert(strmatch(safi2str(SAFI_CRYPTO_ROUTES), "crypto-routes"));
{
	afi_t a; safi_t s;
	assert(bgp_map_afi_safi_iana2int(IANA_AFI_IPV4, IANA_SAFI_CRYPTO_ROUTES, &a, &s) == 0);
	assert(a == AFI_IP && s == SAFI_CRYPTO_ROUTES);
}
assert(afindex(AFI_IP,  SAFI_CRYPTO_ROUTES) == BGP_AF_IPV4_CRYPTO_ROUTES);
assert(afindex(AFI_IP6, SAFI_CRYPTO_ROUTES) == BGP_AF_IPV6_CRYPTO_ROUTES);
assert(afindex(AFI_L2VPN, SAFI_CRYPTO_ROUTES) == BGP_AF_MAX);

/* test_mp_attr.c — add a parse_test[] entry: MP_REACH with AFI=1, SAFI=241,
   4-byte nexthop, one /24 prefix → expect SHOULD_PARSE, 1 NLRI. */
```

### Topotest — 🆕 `tests/topotests/bgp_crypto_routes/`

```
bgp_crypto_routes/
├── __init__.py
├── test_bgp_crypto_routes.py
├── r1/{bgpd.conf,zebra.conf}       # AS65001, originates crypto-routes
├── r2/{bgpd.conf,zebra.conf}       # AS65002, transit + inbound route-map
└── r3/{bgpd.conf,zebra.conf}       # AS65003, receiver; also unicast 192.0.2.0/24
```

Full topology + 11 assertions in [`VALIDATION_PLAN.md` §5](./VALIDATION_PLAN.md#5-integration-topotest) and [`EXECUTION_TESTING_PLAN.md` §5](./EXECUTION_TESTING_PLAN.md).

---

## 11. Exhaustive `switch`/`if` Site Checklist (verified against `bc8971168b`)

`-Wswitch` + `-Werror` will stop the build until each is done:

| File | Line(s) | Site | Action |
|---|---|---|---|
| `lib/prefix.c` | ~180 | `safi2str()` | ➕ `"crypto-routes"` |
| `lib/iana_afi.h` | ~91, ~119 | `safi_iana2int`, `safi_int2iana` | ➕ 241 ↔ enum |
| `bgpd/bgpd.h` | ~3016, ~3040, ~3059, ~3076 | `afindex()` ×4 | ➕ arms |
| `bgpd/bgp_open.c` | ~222, ~282 | `bgp_capability_vty_out()` ×2 | ➕ display arms |
| `bgpd/bgp_attr.c` | ~4945, ~4995, ~5208, ~5333 | nexthop ×2, prefix, prefix-size | ➕ join unicast arm |
| `bgpd/bgp_packet.c` | ~317 | `bgp_nlri_parse()` | ➕ → `bgp_nlri_parse_ip` |
| `bgpd/bgp_route.c` | ~16029 | `bgp_rd_from_dest()` | ➕ `return NULL` |
| `bgpd/bgp_vty.c` | ~176, ~199, ~643, ~667, ~691, ~708 | `bgp_node_type` ×2, `get_bgp_default_af_flag` ×4 | ➕ arms |
| `bgpd/rfapi/rfapi_import.c` | ~205, ~3823, ~4068 | fall-through groups | ➕ no-op arm |
| `bgpd/rfapi/rfapi_monitor.c` | ~188, ~265 | fall-through groups | ➕ no-op arm |

**Non-switch `if`/OR chains** (won't break build, but wrong behaviour if skipped):
`bgpd/bgpd.h` `peer_afi_active_nego()`, `peer_group_af_configured()`; `bgpd/bgpd.c` `bgp_create()` maxpaths.

**`if (safi == SAFI_UNREACH)` guards that crypto-routes deliberately does NOT join** (v1 = no NHT / no GR-NSF / no zebra sync): `bgp_nht.c:~1400`, `bgp_fsm.c:~2902`, `bgp_zebra.c:~5014`.

Enumerate before starting:
```console
grep -rn "case SAFI_UNREACH:" bgpd/ lib/
grep -rn "switch (safi)\|switch (packet->safi)" bgpd/ lib/
```

---

## 12. Deliberately NOT Changed

| Area | Reason |
|---|---|
| `zebra/`, ZAPI, `lib/zclient.c` per-value | v1 control-plane only — nothing sent to zebra |
| `bgp_zebra.c` FIB gates | crypto-routes never installed to FIB |
| `bgp_nht.c` NHT registration | informational nexthop, no tracking in v1 |
| `bgp_update()` / `bgp_nlri_parse_ip()` signatures | no out-of-band NLRI data (unlike `SAFI_UNREACH`'s Reporter TLV) |
| `struct bgp_path_info_extra` | no per-path crypto state |
| `FOREACH_AFI_SAFI_NSF` | no GR/NSF in v1 |
| `bgpd/subdir.am` | no new `.c` file |
| `configure.ac` | no new build option |
| `mgmtd/`, `yang/` | no northbound in v1 (bgpd config is DEFUN-based) |
| all non-BGP daemons | SAFI is a BGP concept; `SAFI_MAX` bump only recompiles them |

---

## 13. Commit Order & Buildability

Each commit compiles (`-Werror` clean) and passes `make check`; the series is bisectable per FRR workflow.

| Commit | Buildable | Testable after |
|---|---|---|
| 1 lib + rfapi | ✅ | unit: mapping/str |
| 2 bgpd.h/c af_index + RIB | ✅ | unit: `afindex()` |
| 3 capability + attr + packet | ✅ | unit: `test_mp_attr` MP_REACH SAFI 241 |
| 4 route processing | ✅ | — |
| 5 CLI + vtysh | ✅ | manual: `address-family ipv4 crypto-routes`, `neighbor activate`, capability negotiation |
| 6 show | ✅ | manual: `show bgp ipv4 crypto-routes` |
| 7 debug | ✅ | manual: `debug bgp crypto-routes` |
| 8 doc | n/a | `make -C doc` |
| 9 tests | n/a | full topotest + regression |

Every commit message: `dir: imperative ≤50-char subject`, body wrapped 72, ending `Signed-off-by: <real name> <email>`.
