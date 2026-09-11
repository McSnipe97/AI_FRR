# FRRouting — Session Prompts, Actions & Repository Analysis

> Deliverable for **Prompt 1** of the CISCO AI assignment (`ProblemStatment.md`): build a developer foundation for introducing a new BGP address family (AFI/SAFI) `crypto_routes` into FRRouting.
> The full technical analysis is identical to [`REPO_ANALYSIS_LOG.md`](./REPO_ANALYSIS_LOG.md); it is reproduced below so this file is self-contained.

---

## Prompts Used This Session

### Prompt 1 (executed here)

> - Perform a comprehensive analysis of this repository to build a developer foundation for future modifications.
> - Identify the core programming languages, frameworks, and primary dependencies.
> - Map out the directory structure and explain the purpose of key folders and entry points.
> - Explain the data flow and how the main components interact with each other.
> - Detail the prerequisites and exact steps needed to spin up the local development environment.
> - Outline the existing testing suites and how to run them.
> - Highlight any specific coding standards, patterns, or architecture rules enforced in this codebase.
> - Write all of the above sections, along with a summary of the prompts used and actions taken during this session, into `ClaudePromptsAndReplies.md`.
> - Ensure the output is cleanly formatted in Markdown with proper headers, code blocks, and bullet points.
> - Save all your findings, structural insights, and a log of this analysis into a new file named `REPO_ANALYSIS_LOG.md`.

### Prompt 2 (not yet executed)

> From Analysis, Build a High Level Design Document v1 (in markdown) & a Low Level Design Document v1 (in markdown).

### Prompt 3 (not yet executed)

> Introduce a new BGP address family (AFI/SAFI) named `crypto_routes`. Document any code changes required. Changes to be reviewed before compilation.

## Actions Taken This Session

1. Read the assignment docs (`ProblemStatment.md`, `PromptsAndReplies.md`, `.gitmodules`) — established that the repo wraps the FRRouting source as a submodule and the objective is a new BGP SAFI.
2. Enumerated the repository and the `frr/` submodule tree (`ls`, `git log`) — confirmed submodule checkout `frr-10.8.0-dev-1194-gbc8971168b`.
3. Read FRR developer docs: `README.md`, `building-frr-for-ubuntu2x04.rst`, `include-compile.rst`, `process-architecture.rst`, `testing.rst`, `topotests.rst`, `workflow.rst`, `checkpatch.rst`, `cli.rst`.
4. Inspected the enforced toolchain config: `.clang-format`, `.flake8`, `.pylintrc`, `.isort.cfg`, `.git-blame-ignore-revs`.
5. Inspected `tests/` (unit-test dirs, `pytest.ini`, `subdir.am`) and the `configure.ac` feature-flag surface.
6. Read the AFI/SAFI type definitions (`lib/zebra.h`, `lib/iana_afi.h`) and mapped the `bgpd/` file set to identify every touch point for adding a SAFI.
7. Wrote `REPO_ANALYSIS_LOG.md` and this file.
8. **No build was run and no source was modified.** HLD/LLD and the implementation are Prompts 2 and 3.

---


**Analysis date:** 2026-09-10
**Repo:** `/Users/mcsnipe97/workbench/Interviews/Cisco/AI_FRR`
**Subject:** `frr/` git submodule → `https://github.com/FRRouting/frr.git`
**Submodule checkout:** `bc8971168b` — `frr-10.8.0-dev-1194-gbc8971168b` (version string `10.8.0-dev`, from `configure.ac` `AC_INIT`)
**Context:** Assignment repo. `ProblemStatment.md` sets the eventual goal — *introduce a new BGP address family (AFI/SAFI) named `crypto_routes`*. This document is the analysis foundation for that change (Prompt 1); HLD/LLD and implementation are later prompts.

---

## 1. Repository Layout (this repo, not FRR)

```
AI_FRR/
├── .gitmodules              # single submodule: frr → github.com/FRRouting/frr
├── .gitignore               # ignores .DS_Store
├── ProblemStatment.md       # the assignment: add BGP AFI/SAFI "crypto_routes"
├── PromptsAndReplies.md      # running prompt log (Prompts 1–3)
├── ClaudePromptsAndReplies.md# this analysis + session log (Prompt 1 deliverable)
├── REPO_ANALYSIS_LOG.md      # this file (Prompt 1 deliverable)
└── frr/                      # FRRouting source tree (submodule)
```

Everything below concerns `frr/`.

---

## 2. Core Languages, Frameworks & Dependencies

### Languages

| Language | Where | Purpose |
|---|---|---|
| **C (GNU C11)** | `lib/`, all `*d/` daemon dirs, `zebra/` | The routing engine and every protocol daemon. ~95% of the codebase. |
| **C++** | `grpc/`, `lib/northbound_grpc.cpp`, a few tests | Only the optional gRPC northbound plugin. |
| **Python 3** | `tests/topotests/`, `python/`, `tools/`, `*_clippy` codegen | Integration test framework, build-time code generation, tooling. |
| **YANG 1.1** | `yang/` | Data models for the `mgmtd` / northbound configuration path. |
| **Protocol Buffers** | `*.proto` in `fpm/`, `qpb/`, `grpc/` | FPM (Forwarding Plane Manager) and gRPC wire formats. |
| **Lua** | `lib/frrlua.c`, `lib/frrscript.c` | Optional embedded scripting hooks (route-maps, etc.). |
| **Lex / Yacc (Bison/Flex)** | `lib/command_lex.l`, `lib/command_parse.y` | The CLI command-graph parser. |
| **M4 / Autoconf** | `configure.ac`, `m4/` | Build configuration. |
| **Shell** | `bootstrap.sh`, `tools/`, `*.sh` | Build bootstrap, packaging, helper scripts. |

### Build system

- **GNU Autotools**: `autoconf` + `automake` + `libtool`.
- **Non-recursive automake**: one top-level `Makefile.am` that `include`s a `subdir.am` from each component directory. There is **no per-directory Makefile.am you can build in isolation** — always build from the tree root.
- `bootstrap.sh` → `autoreconf` → generates `configure`.
- `configure.ac` is ~92 KB and drives ~100 `--enable-*` / `--with-*` feature toggles (per-daemon enable flags, `--enable-grpc`, `--enable-snmp`, `--enable-config-rollbacks`, `--enable-multipath=N`, `--enable-address-sanitizer`, …).

### Primary external dependencies (build-time)

| Dependency | Required? | Used for |
|---|---|---|
| `libyang2` (≥ 2.x) | **Yes** | YANG parsing for northbound/mgmtd. Often built from source (`doc/developer/building-libyang.rst`). |
| `libjson-c` | Yes | JSON output for all `show ... json` CLI commands and topotest assertions. |
| `libreadline` | Yes | `vtysh` interactive shell. |
| `bison`, `flex` | Yes | CLI parser generation. |
| `libc-ares` | Yes | Async DNS resolution. |
| `libelf`, `libunwind` | Yes (Linux) | ELF xref section parsing, backtraces. |
| `libcap` | Yes (Linux) | POSIX capabilities / privilege dropping. |
| `protobuf-c` (`libprotobuf-c`, `protobuf-c-compiler`) | Yes | FPM protobuf, qpb. |
| `libpam` | Optional | `vtysh` PAM auth. |
| `net-snmp` (`libsnmp-dev`) | Optional (`--enable-snmp`) | AgentX SNMP subagent. |
| `libgrpc++`, `grpc` plugin | Optional (`--enable-grpc`) | gRPC northbound. |
| `libsqlite3` | Optional (`--enable-config-rollbacks`) | Config rollback history DB. |
| `libzmq` | Optional (`--enable-zeromq`) | ZeroMQ event bus. |
| `libpcre2` / `libpcreposix` | Optional | Regex in route-maps / prefix-lists. |
| Python `sphinx` | Doc build only | `doc/` → HTML/man. |

### Vendored / in-tree

- `pceplib/` — PCEP protocol library (for `pathd`), builds as part of the tree.
- `lib/` itself is the shared `libfrr` linked by every daemon.

---

## 3. Directory Structure & Entry Points

### The two-layer model

FRR is **not one process**. It is a suite of single-purpose daemons that share one library (`libfrr`, built from `lib/`) and talk to a central RIB manager (`zebra`) over a Unix-domain socket protocol called **ZAPI**.

```
        vtysh (CLI multiplexer)              mgmtd (YANG/northbound front-end)
             │  (per-daemon vty socket)            │  (mgmt back-end socket)
   ┌─────────┼───────────┬───────────┬─────────────┼──────────┐
 bgpd      ospfd       isisd       staticd       pimd   ...  (protocol daemons)
   │          │           │           │            │
   └──────────┴───────────┴─────ZAPI──┴────────────┘
                          │
                        zebra   ── RIB, nexthop resolution, interface state
                          │
                 ┌────────┴────────┐
             kernel (netlink)   dplane plugins (FPM, dplane_fpm_nl, ...)
```

### Key top-level directories

| Path | Purpose |
|---|---|
| `lib/` | **`libfrr`** — the shared core. Event loop (`event.c`, `frrevent.h`), CLI/command graph (`command.c`, `command_parse.y`), VTY (`vty.c`), ZAPI client (`zclient.c`/`.h`), northbound framework (`northbound*.c`), prefix/AFI-SAFI types (`prefix.h`, `zebra.h`, `iana_afi.h`), route-maps (`routemap.c`), prefix-lists (`filter.c`, `plist.c`), nexthop (`nexthop.c`), typesafe containers (`typesafe.h`), memory (`memory.c`), logging (`zlog`). ~295 files. **Every cross-daemon concept lives here.** |
| `zebra/` | The RIB / kernel-interface daemon. `main.c` entry, `zebra_rib.c` (RIB), `zapi_msg.c` (ZAPI server side), `rt_netlink.c` (Linux FIB programming), `zebra_dplane.c` (async dataplane abstraction), `redistribute.c`. |
| `bgpd/` | BGP daemon — **the daemon the assignment touches**. 144 files. Entry `bgp_main.c`. See §5 for the file map. |
| `ospfd/`, `ospf6d/`, `isisd/`, `ripd/`, `ripngd/`, `pimd/`, `pim6d/`, `ldpd/`, `eigrpd/`, `babeld/`, `nhrpd/`, `vrrpd/` | Individual routing/multicast protocol daemons, each with a `*_main.c`. |
| `staticd/` | Static route daemon (static routes are not in zebra itself). |
| `pathd/` | Segment-routing traffic-engineering (uses `pceplib/`). |
| `pbrd/` | Policy-based routing. |
| `sharpd/` | "Super Happy Advanced Routing Process" — a test/traffic-injection daemon used heavily by topotests. |
| `mgmtd/` | Centralized management daemon: applies YANG config to daemons via the northbound back-end API. Migration from per-daemon config is in progress (`staticd` fully migrated; most others not). |
| `vtysh/` | The integrated shell. Connects to each daemon's vty socket; `extract.pl` scrapes `DEFUN`/`DEFPY` from all daemons to build its combined command set. |
| `watchfrr/` | Supervises/restarts daemons; part of the `vtysh`/service integration. |
| `lib/` + `yang/` | `yang/` holds the `.yang` models; `lib/northbound*` is the framework that binds YANG paths to C callbacks. |
| `tools/` | `frr-reload.py`, init scripts, `etc/frr/` sample configs (`daemons`, `frr.conf`, `vtysh.conf`), `checkpatch` wrappers. |
| `tests/` | Unit tests (`tests/<daemon>/test_*.c`) + integration tests (`tests/topotests/`). See §6. |
| `doc/` | `doc/user/` (user guide), `doc/developer/` (dev guide — **read `workflow.rst`, `process-architecture.rst`, `cli.rst`, `northbound/`, `testing.rst`, `topotests.rst`, `building*.rst`**), `doc/manpages/`. |
| `include/` | Kernel/system header shims. |
| `fpm/`, `qpb/`, `grpc/` | FPM protobuf defs, protobuf helpers, gRPC plugin. |
| `debian/`, `redhat/`, `alpine/`, `snapcraft/`, `pkgsrc/`, `docker/` | Packaging. |
| `m4/` | Autoconf macros. |
| `gdb/` | GDB pretty-printers / helper macros for FRR data structures. |
| `python/` | Build-time codegen: `clidef.py` (clippy — expands `DEFPY`), `xrelfo.py` (xref/ELF), `makevars`. |

### Per-daemon entry point pattern

Every daemon's `*_main.c` does the same dance:

1. `frr_preinit(&<daemon>_di, argc, argv)` — set up the daemon info struct.
2. Parse daemon-specific options via `frr_opt` / `frr_getopt`.
3. `frr_init()` — creates the `event_loop` ("threadmaster"), reads config, opens the vty socket, connects `zclient` to zebra.
4. Protocol-specific init (e.g. `bgp_master_init`, `bgp_init`).
5. `frr_run(master)` — enters the event loop. Runs until SIGINT.

---

## 4. Data Flow & Component Interaction

### 4.1 Configuration in (two paths, coexisting)

**Legacy / CLI path:**
```
operator types in vtysh
  → vtysh forwards line to the owning daemon's VTY socket
  → daemon's command graph (built from DEFUN/DEFPY via lib/command.c) matches it
  → the DEFUN handler mutates in-memory protocol state
```

**Northbound / YANG path (mgmtd):**
```
config (CLI in mgmtd, or gRPC/sysrepo)
  → mgmtd validates against YANG model (libyang)
  → mgmt transaction → mgmt back-end socket → daemon's northbound callbacks
  → nb_callbacks (create/modify/destroy/apply_finish) mutate protocol state
```
Only some daemons are retrofitted. `staticd` is the reference implementation; `bgpd` has partial northbound coverage (route-maps, some knobs) but its core BGP config is still DEFUN-based.

### 4.2 Routing information flow (BGP as the example)

```
peer TCP:179  ──► bgp_packet.c (read task on the event loop)
                    │  parse OPEN/UPDATE/KEEPALIVE/NOTIFICATION
                    ▼
              bgp_attr.c   parse path attributes (incl. MP_REACH_NLRI / MP_UNREACH_NLRI → per-AFI/SAFI NLRI)
                    │
                    ▼
              bgp_route.c  bgp_update() — run inbound route-map/filters, build bgp_path_info,
                           install into the per-(AFI,SAFI) bgp_table (bgp_table.c, a route_table trie)
                    │
                    ▼
              bgp_route.c  bgp_best_selection() — best-path algorithm per prefix
                    │
        ┌───────────┴───────────────┐
        ▼                           ▼
 bgp_updgrp_*.c                 bgp_zebra.c
 outbound update generation      bgp_zebra_announce() → ZAPI ZEBRA_ROUTE_ADD
 (per update-group, outbound      to zebra for the winning path
  route-map), send via bgp_packet
```

### 4.3 zebra: RIB → FIB

```
ZAPI ZEBRA_ROUTE_ADD (from bgpd/ospfd/staticd/...)
  → zapi_msg.c decodes → zebra_rib.c rib_add()
  → RIB run: administrative-distance contest between protocols per prefix
  → nexthop resolution (recursive nexthops resolved against the RIB)
  → winning route handed to zebra_dplane.c (async queue, own pthread)
  → dataplane provider: rt_netlink.c programs the Linux FIB via netlink
    (and/or FPM/dplane_fpm_nl.c streams routes to an external forwarding agent)
  → results (success/fail) flow back up asynchronously
```

zebra also pushes **down** to daemons: interface add/delete/up/down, address changes, redistribution, nexthop-tracking (NHT) updates — all over the same ZAPI socket, consumed by each daemon's `zclient` callbacks.

### 4.4 The event loop (every daemon)

Single-threaded cooperative event loop per pthread (`lib/event.c`, struct `event_loop` aka "threadmaster"). Task types: `EVENT_READ`, `EVENT_WRITE`, `EVENT_TIMER`, `EVENT_EVENT` (high-priority, used to drive protocol FSMs), plus internal ones. `event_add_*()` schedules; `event_fetch()` + `event_call()` in `frr_run()` dispatch. A few daemons (bgpd, zebra) use additional pthreads (e.g. bgpd's I/O and keepalive pthreads, zebra's dplane pthread), each with its own loop; cross-thread work is posted, not shared-locked.

### 4.5 AFI/SAFI — the axis the assignment extends

Defined in `lib/zebra.h`:

```c
typedef enum { AFI_UNSPEC=0, AFI_IP=1, AFI_IP6=2, AFI_L2VPN=3, AFI_BGP_LS=4, AFI_MAX=5 } afi_t;
typedef enum { SAFI_UNSPEC=0, SAFI_UNICAST=1, SAFI_MULTICAST=2, SAFI_MPLS_VPN=3,
               SAFI_ENCAP=4, SAFI_EVPN=5, SAFI_LABELED_UNICAST=6, SAFI_FLOWSPEC=7,
               SAFI_BGP_LS=8, SAFI_UNREACH=9, SAFI_MAX=10 } safi_t;
```

- `lib/iana_afi.h` maps these internal enums ↔ IANA wire numbers (used in BGP capability negotiation and MP_REACH/MP_UNREACH).
- `FOREACH_AFI_SAFI(afi, safi)` in `lib/zebra.h` is the iteration macro used all over bgpd; **`SAFI_MAX` sizes many static arrays** (`bgp->rib[AFI_MAX][SAFI_MAX]`, peer AF config arrays, etc.).
- Adding a SAFI therefore is not a localized change: it ripples through `bgpd/` (tables, capabilities, NLRI encode/decode, best-path, zebra announce, CLI `address-family` nodes, route-map/filter attach points), `lib/` (the enum, `iana_afi.h`, `afi2str`/`safi2str`, zclient SAFI plumbing), and `zebra/` (if the routes are to be installed in the kernel RIB).

---

## 5. `bgpd/` File Map (for the crypto_routes change)

| File | Role |
|---|---|
| `bgp_main.c` | Entry point, daemon init. |
| `bgpd.h` / `bgpd.c` | Core structs: `struct bgp`, `struct peer`, `struct peer_af`, global `bm` (bgp_master). AF-enable bitmaps. |
| `bgp_open.c` / `bgp_open.h` | Capability negotiation — **Multiprotocol Extensions capability carries the (AFI,SAFI) pair**; new SAFI must be advertised/parsed here. |
| `bgp_attr.c` / `bgp_attr.h` | Path attribute parse/encode, incl. `MP_REACH_NLRI` (14) / `MP_UNREACH_NLRI` (15) with AFI/SAFI headers. |
| `bgp_packet.c` | UPDATE assembly/parsing, calls into NLRI handlers per AFI/SAFI. |
| `bgp_nlri.c` (in `bgp_route.c`/`bgp_mp_route`-style handlers) | NLRI prefix (de)serialization; a new SAFI needs an encode/decode path or an explicit "reuse unicast format" decision. |
| `bgp_route.c` / `bgp_route.h` | `bgp_update()`, best-path (`bgp_best_selection`), the per-AF `bgp_table`, `bgp_static_*`, `bgp_aggregate_*`. `bgp_node`/`bgp_dest` per prefix. |
| `bgp_table.c` / `bgp_table.h` | `struct bgp_table` wrapper over `lib/table.c` route trie; one per (AFI,SAFI) per `struct bgp`. |
| `bgp_updgrp*.c` | Update-group / subgroup outbound advertisement machinery (must be AFI/SAFI-aware). |
| `bgp_zebra.c` / `bgp_zebra.h` | ZAPI glue: `bgp_zebra_announce/withdraw`, redistribution, `bgp_zebra_instance_register`. Maps BGP (AFI,SAFI) → zebra route type & AFI. |
| `bgp_vty.c` | The giant CLI file: `address-family <afi> <safi>` command nodes, `neighbor X activate`, per-AF knobs. New `address-family` node wiring goes here. |
| `bgp_route.c` + `bgp_attr.c` + `bgp_aspath.c`/`bgp_community*.c` | Attribute handling shared across AFs. |
| `bgp_fsm.c` | Peer finite-state machine (driven by `EVENT_EVENT` tasks). |
| `bgp_nexthop.c` / `bgp_nht.c` | Nexthop tracking / validation per AF. |
| `bgp_rd.c`, `bgp_mplsvpn.c`, `bgp_evpn*.c`, `bgp_flowspec*.c` | **Precedents** — each is "how a non-unicast SAFI was added". `bgp_flowspec*` and `bgp_evpn*` are the closest templates for introducing a brand-new SAFI end-to-end. |
| `bgp_debug.c` | Per-AF debug toggles. |
| `bgp_snmp*.c`, `bgp_bmp.c` | SNMP MIB and BGP Monitoring Protocol — may need AF awareness (lower priority for a v1). |

**Recommended template:** study `git log --follow bgpd/bgp_flowspec.c` and the commits that introduced `SAFI_FLOWSPEC` — it is the most recent full "new SAFI" example and shows every touch point.

---

## 6. Testing

### 6.1 Unit tests (C, TAP, fast, no root)

- Location: `tests/<component>/test_*.c` with a sibling `test_*.py` wrapper (the `.py` is a thin `frrtest` harness that runs the C binary and checks TAP output; some have a `.refout` golden file).
- Coverage: `tests/lib/` (containers, prefix, nexthop, table, printfrr, zlog, graph, typelist, …), `tests/bgpd/` (`test_aspath`, `test_attr_parse`, `test_bgp_table`, `test_capability`, `test_mp_attr`, `test_community`, `test_ecommunity`, `test_peer_attr`, `test_packet`), `tests/zebra/`, `tests/ospfd/`, `tests/ospf6d/`, `tests/isisd/`.
- Wired into automake via `tests/subdir.am` (`check_PROGRAMS`, `TESTS`).
- **Run:**
  ```console
  ./bootstrap.sh && ./configure <opts> && make
  make check          # runs the whole TAP suite; on *BSD use gmake check
  # single test:
  ./tests/bgpd/test_mp_attr
  python3 ./tests/bgpd/test_mp_attr.py
  ```
- `tests/bgpd/test_mp_attr.c` and `test_capability.c` are directly relevant to a new AFI/SAFI — they exercise MP_REACH/MP_UNREACH and MP capability parsing.

### 6.2 Topotests (Python, integration, **root required**)

- Framework: `pytest` on top of *micronet* (network-namespace based virtual topologies). Located in `tests/topotests/` (~587 test dirs). Config: `tests/pytest.ini` (`norecursedirs = topotests` keeps `make check` out of them).
- Requirements (`doc/developer/topotests.rst`):
  ```console
  apt-get install gdb iproute2 net-tools python3-pip iputils-ping iptables tshark valgrind ssmping
  python3 -m pip install wheel 'pytest>=8.3.2' 'pytest-asyncio>=0.24.0' 'pytest-xdist>=3.6.1' \
      'scapy>=2.4.5' 'libyang<4' pyyaml xmltodict 'protobuf<4'
  python3 -m pip install git+https://github.com/Exa-Networks/exabgp@0659057837cd6c6351579e9f0fa47e9fb7de7311
  useradd -d /var/run/exabgp/ -s /bin/false exabgp
  ```
  Tested on Ubuntu 22.04/24.04, Debian 12/13. ExaBGP only needed for BGP tests.
- The build must be installed (or symlinked) where topotest expects daemons — typically `--prefix=/usr --sysconfdir=/etc --localstatedir=/var --enable-multipath=64` then `make install`, plus symlink `vtysh` into `/usr/lib/frr/`.
- **Run:**
  ```console
  cd tests/topotests
  sudo -E python3 -m pytest -s -v                       # everything
  sudo -E pytest -s -v -nauto --dist=loadfile           # parallel
  sudo -E pytest bgp_ipv4_over_ipv6/test_*.py           # one suite
  ```
- Logs land in `/tmp/topotests/`; `tests/topotests/analyze.py` post-processes `topotests.xml`.
- Options: `--valgrind-memleaks`, `--asan-abort`, `--pcap=all`, `--pause-on-error`, `--vtysh`/`--gdb-routers` for interactive debugging.
- For `crypto_routes`, the deliverable would be a new `tests/topotests/bgp_crypto_routes/` dir with a `test_bgp_crypto_routes.py`, router `.conf` files, and JSON reference outputs, following e.g. `bgp_flowspec/` or `bgp_l3vpn_to_bgp_vrf/`.

### 6.3 CI

- `.github/workflows/` (github-ci) runs style checks, the unit suite, and a large topotest matrix. `.travis.yml` present but legacy.
- Style bot ("frrbot") enforces `clang-format` and `checkpatch` on PRs.

---

## 7. Coding Standards & Architecture Rules (enforced)

### Style (mechanically checked)

- **`clang-format`** (`.clang-format`, needs clang-format ≥ 11). Linux-kernel-derived: **tabs for indent, 8-wide**, ~80 col target (100 hard), `AfterFunction: true` brace wrapping (opening brace on its own line for function definitions, K&R elsewhere), `AlignConsecutiveMacros: true`, pointer binds right (`char *p`). `git clang-format` on your diff before submitting.
- **`scripts/checkpatch.pl`** — kernel checkpatch, adapted. "Checkpatch is not always right; your judgement takes precedence" (`doc/developer/checkpatch.rst`), but CI flags every hit.
- **Python:** `.flake8`, `.pylintrc`, `.isort.cfg` — `black`-style formatting for topotests/tooling.
- `.git-blame-ignore-revs` exists — bulk reformat commits are excluded from blame.

### Source file rules (`doc/developer/workflow.rst` §"Coding Practices & Style")

- **SPDX header required** in every `.c/.h/.cpp/.py`: `// SPDX-License-Identifier: GPL-2.0-or-later` (no boilerplate). `.yang`/`.proto` need SPDX **and** full boilerplate. New files need a `Copyright (C) YEAR Author` line. **Never delete a Copyright/author line.** When modifying an existing file substantially, add a `Portions:` copyright entry.
- **First include** in any `.c` **must** be `<zebra.h>` or `"config.h"` (HAVE_CONFIG_H-guarded). Then, blank-line-separated groups: system headers, `lib/` headers, daemon headers, then (headers only) `extern "C" {` and forward decls. Use `#include "lib/foo.h"` (full path from tree root), not `<...>`, except `<zebra.h>`.
- `zebra.h` is an "include multiplexer" for system headers — do **not** add FRR declarations to it.

### Defensive coding (PRs rejected otherwise)

- **`strcpy` / `strcat` / `sprintf` are banned, no exceptions.** Use `strlcpy` / `strlcat` / `snprintf`.
- Size args to `strlcpy`/`snprintf` **must** use `sizeof(...)`, not a length constant.
- Zero-init stack structs/arrays with `struct foo x = {};` initializer expressions, **not** `memset()` (avoids missed branches / wrong size). But do **not** zero-init a value that must start non-zero — let the compiler/ASAN catch uninitialized use.
- **`system()`, `fork()`, `exec*()` are disallowed** in daemon code — they break signal-driven shutdown.
- CERT/MISRA "may provide useful input" but are not applied as-is.

### Data-structure rules

- **New code and refactors must use the typesafe containers** in `lib/typesafe.h` (`DECLARE_LIST`, `DECLARE_RBTREE_UNIQ`, `DECLARE_HASH`, `DECLARE_DLIST`, …), documented in `doc/developer/lists.rst`. Old `lib/linklist.h` / `openlist` style is legacy-only.
- Route storage is a radix trie: `lib/table.c` (`struct route_table` / `route_node`), wrapped per-daemon (`bgp_table.c`).
- Memory: every allocation goes through `XMALLOC`/`XCALLOC`/`XFREE` with a `DEFINE_MTYPE`/`DEFINE_MTYPE_STATIC` memory type — enables `show memory` accounting and leak detection. No bare `malloc`.
- **RCU**: `lib/frrcu.*` — threads must hold an RCU read lock; see `doc/developer/rcu.rst` and `locking.rst`.

### CLI rules (`doc/developer/cli.rst`)

- **New commands use `DEFPY`** (clippy-preprocessed, auto-typed args), not `DEFUN`. Register with `install_element(NODE, &foo_cmd);`.
- Command strings are a graph grammar parsed by `lib/command_parse.y`; `lib/clippy` (built early) generates `*_clippy.c` from `DEFPY` at build time.
- `vtysh/extract.pl` harvests every daemon's DEFUNs to build the unified `vtysh` — a new command is automatically reachable from `vtysh` once it builds.
- Config-writing (`show running-config`) is a separate callback per node — new config state needs a matching write function.

### Northbound rules (`doc/developer/northbound/`)

- New *configuration* should ideally be modeled in YANG (`yang/frr-*.yang`) with `nb_callbacks` rather than raw DEFUN state mutation, following the `retrofitting-configuration-commands.rst` guide. In practice bgpd is only partly converted, so a v1 `crypto_routes` can follow the existing bgpd DEFUN pattern and note northbound as follow-up.

### Workflow rules

- **DCO / `Signed-off-by` required** on every commit (real name + email; no anonymous/pseudonymous). No sign-off = not merged.
- Commit message format = Linux kernel style: `dir: short summary` (≤ 50 char subject, e.g. `bgpd: add crypto_routes address family`), blank line, prose body wrapped at 72. Commit messages that are only program output are rejected.
- One logical change per commit; series must be bisectable (every commit builds).
- PRs on GitHub; maintainers + CI (style, unit, topotest) must pass.
- Backports to `stable/*` branches are cherry-picked and must flow to all newer stable branches too.

---

## 8. Local Dev Environment — Exact Steps

Reference: `doc/developer/building-frr-for-ubuntu2x04.rst` + `include-compile.rst`. Ubuntu 22.04/24.04 or Debian 12/13 recommended (matches CI + topotests).

### 8.1 Dependencies (Ubuntu 24.04)

```console
sudo apt update
sudo apt-get install \
   git autoconf automake libtool make libreadline-dev texinfo \
   pkg-config libpam0g-dev libjson-c-dev bison flex \
   libc-ares-dev python3-dev python3-sphinx \
   install-info build-essential libsnmp-dev perl \
   libcap-dev libelf-dev libunwind-dev \
   protobuf-c-compiler libprotobuf-c-dev

# optional extras
sudo apt-get install libgrpc++-dev protobuf-compiler-grpc   # --enable-grpc
sudo apt-get install libsqlite3-dev                         # --enable-config-rollbacks
sudo apt-get install libzmq5 libzmq3-dev                    # --enable-zeromq
```

### 8.2 libyang (usually build from source — distro version often too old)

Follow `doc/developer/building-libyang.rst`:

```console
sudo apt-get install cmake libpcre2-dev
git clone https://github.com/CESNET/libyang.git
cd libyang && git checkout v2.1.148   # a known-good v2.x tag
mkdir build && cd build
cmake -D CMAKE_INSTALL_PREFIX:PATH=/usr -D CMAKE_BUILD_TYPE:String="Release" ..
make && sudo make install && sudo ldconfig
```

### 8.3 Users/groups (needed only for `make install` / running daemons)

```console
sudo groupadd -r -g 92 frr
sudo groupadd -r -g 85 frrvty
sudo adduser --system --ingroup frr --home /var/run/frr/ \
   --gecos "FRR suite" --shell /sbin/nologin frr
sudo usermod -a -G frrvty frr
```

### 8.4 Build (from the submodule root: `AI_FRR/frr/`)

```console
cd frr
git submodule update --init --recursive     # if pceplib etc. missing
./bootstrap.sh
./configure \
    --prefix=/usr \
    --includedir=\${prefix}/include \
    --bindir=\${prefix}/bin \
    --sbindir=\${prefix}/lib/frr \
    --libdir=\${prefix}/lib/frr \
    --libexecdir=\${prefix}/lib/frr \
    --sysconfdir=/etc \
    --localstatedir=/var \
    --with-moduledir=\${prefix}/lib/frr/modules \
    --enable-multipath=64 \
    --enable-user=frr --enable-group=frr --enable-vty-group=frrvty \
    --enable-configfile-mask=0640 --enable-logfile-mask=0640 \
    --enable-snmp \
    --with-pkg-git-version --with-pkg-extra-version=-crypto-routes-dev
make -j$(nproc)
```

For a **fast inner dev loop** (no install, run daemons from the build tree):

```console
./configure --enable-dev-build --enable-address-sanitizer --prefix=/usr --localstatedir=/var
make -j$(nproc)
make check                       # unit tests
```

`--enable-dev-build` turns on extra asserts and debug; pair with `--enable-address-sanitizer` while developing the new SAFI.

### 8.5 Install + config (only if you want to run a live router / topotests)

```console
sudo make install
sudo install -m 775 -o frr -g frr -d /var/log/frr
sudo install -m 775 -o frr -g frrvty -d /etc/frr
sudo install -m 640 -o frr -g frrvty tools/etc/frr/vtysh.conf /etc/frr/vtysh.conf
sudo install -m 640 -o frr -g frr tools/etc/frr/{frr.conf,daemons,support_bundle_commands.conf} /etc/frr/
# enable bgpd + zebra:
sudo sed -i 's/^bgpd=no/bgpd=yes/' /etc/frr/daemons
sudo sysctl -w net.ipv4.ip_forward=1 -w net.ipv6.conf.all.forwarding=1
sudo systemctl restart frr        # or: /usr/lib/frr/frrinit.sh start
vtysh -c 'show version'
```

### 8.6 Fastest path for *just reviewing code changes before compilation* (this assignment's constraint)

You do **not** need a full runtime to satisfy "changes to be reviewed before compilation":

```console
cd frr
./bootstrap.sh && ./configure --enable-dev-build --disable-doc   # generates headers/clippy
# make edits, then:
git clang-format
./scripts/checkpatch.pl --no-tree -f bgpd/bgp_<file>.c
make -j$(nproc) bgpd/bgpd            # compile just the daemon
make check-TESTS TESTS='tests/bgpd/test_mp_attr'   # targeted unit test
```

---

## 9. Key Risks / Notes for the `crypto_routes` Change

- `SAFI_MAX` bumps array sizes across `struct bgp`, `struct peer`, update-groups, and several `lib/` tables — grep `[SAFI_MAX]` and `[AFI_MAX]` before assuming a change is local (~hundreds of hits).
- Wire interop needs an IANA SAFI code (or a private/experimental one) added to `lib/iana_afi.h` both directions, plus MP capability advertise/parse in `bgp_open.c`.
- NLRI format: decide "identical to unicast prefix encoding" (cheap) vs. a custom TLV (needs encode/decode in the NLRI handlers + `bgp_attr.c` MP_REACH/MP_UNREACH paths).
- Kernel install: if `crypto_routes` should reach the FIB, `bgp_zebra.c` must map it to a zebra AFI/route-type and `zebra` must accept it; if it's control-plane-only (like flowspec/BGP-LS), it stops at bgpd.
- CLI: new `address-family <afi> crypto-routes` node in `bgp_vty.c` + node registration in `lib/command.c` node table + config-write callback.
- Tests: unit test in `tests/bgpd/` for capability + MP attr; topotest dir under `tests/topotests/bgp_crypto_routes/`.
- Follow the `SAFI_FLOWSPEC` / `bgp_flowspec*` introduction commits as the working template.

---

## 10. Analysis Log (what was done this session)

| Step | Action | Tool | Finding |
|---|---|---|---|
| 1 | Read `ProblemStatment.md`, `PromptsAndReplies.md`, `.gitmodules` | Read/Bash | Repo is a wrapper around the FRR submodule; goal is a new BGP AFI/SAFI `crypto_routes`. |
| 2 | Listed repo + `frr/` top-level tree | Bash `ls` | 40+ component dirs; autotools build; submodule at `frr-10.8.0-dev-1194-gbc8971168b`. |
| 3 | Read `frr/README.md`, enumerated `doc/developer/` | Bash | Protocol list, mgmtd/YANG note, doc index. |
| 4 | Read `doc/developer/building-frr-for-ubuntu2x04.rst`, `include-compile.rst` | Bash | Exact apt deps + `bootstrap → configure → make` steps. |
| 5 | Read `doc/developer/process-architecture.rst` | Bash | Event-loop ("threadmaster") model, task types. |
| 6 | Read `doc/developer/testing.rst`, `topotests.rst`; inspected `tests/` tree, `tests/pytest.ini`, `tests/subdir.am` | Bash | Unit tests (`make check`, TAP) vs. topotests (pytest+micronet, root); requirements + run commands. |
| 7 | Read `doc/developer/workflow.rst` "Coding Practices & Style", `checkpatch.rst`, `cli.rst` | Bash | SPDX, include order, banned funcs, typesafe containers, DEFPY, DCO/sign-off, commit format. |
| 8 | Inspected `.clang-format`, `.flake8`, `.pylintrc`, `.isort.cfg`, `.git-blame-ignore-revs`, `.gitignore` | Bash | Enforced formatting toolchain. |
| 9 | Read AFI/SAFI enums in `lib/zebra.h`, `lib/iana_afi.h`; mapped `bgpd/` file list | Bash `grep`/`ls` | Identified every touch point for adding a SAFI; flagged `SAFI_MAX` array-sizing risk; `bgp_flowspec*` as template. |
| 10 | Wrote `REPO_ANALYSIS_LOG.md` and `ClaudePromptsAndReplies.md` | Write | This document. |

**Not done (out of scope for Prompt 1):** no build was run, no code changed, HLD/LLD (Prompt 2) and the `crypto_routes` implementation (Prompt 3) are separate deliverables.

---

## Prompt 2

> - From Analysis, Build a High Level Design Document v1 (in markdown) & a Low Level Design Document v1 (in markdown)
> - Build a reference execution & testing plan for our Problem Statement.
> - Showcase changes in HLDv1 and LLDv1 for our Problem Statement.

### Response 2 — Summary

Produced three documents at the repo root:

| File | Contents |
|---|---|
| `HLD_v1.md` | High-Level Design v1 for the `crypto_routes` SAFI — scope, assumed semantics, design principles, component + data-flow view, (AFI,SAFI) model, NFRs, alternatives considered, risks, phasing, acceptance criteria. **§9 is the HLD-level design-delta showcase** (before/after config + enum, data-flow delta table, change footprint, reference commit decomposition). |
| `LLD_v1.md` | Low-Level Design v1 — file-by-file / function / enum / `switch`-arm change **map** with diff-style snippets, "what is NOT changed and why", build/codegen notes, a **pre-compilation review checklist**, full `switch (safi)` site inventory (Appendix A), "if a new AFI is truly required" (Appendix B). §3 + §6 are the LLD-level showcase. |
| `EXECUTION_TESTING_PLAN.md` | Reference execution & testing plan — env setup, build (with the `-Wswitch` checklist loop), static gates, unit tests to add, full topotest design (`tests/topotests/bgp_crypto_routes/`, 3-router topology, 11 assertions), regression sweep, ASan/leak run, manual smoke, CI expectations, acceptance matrix, rollback, 7-day timeline. |

#### Design decisions (Prompt 2)

- **New SAFI, not new AFI.** `SAFI_CRYPTO_ROUTES = 10` (`SAFI_MAX` 10→11) under existing `AFI_IP`/`AFI_IP6`. A new AFI would resize `[AFI_MAX]` arrays across all of `lib/` and every daemon for no benefit — crypto routes are IP prefixes.
- **Reuse unicast NLRI.** `bgp_nlri_parse()` dispatches `SAFI_CRYPTO_ROUTES` to the existing `bgp_nlri_parse_ip()`. No new wire format, no new parser, no new RIB type, **no `bgp_update()` signature change**.
- **Control-plane only in v1.** Deliberately excluded from the `bgp_zebra` FIB-install gate — like BGP-LS / FlowSpec / `SAFI_UNREACH`.
- **IANA SAFI 241** (Private-Use range 241–254) — no squatting on an unassigned public number; configurable value deferred to v2.
- **Template:** the merged `SAFI_UNREACH` work (15-commit series, `023da6de44 … 1b2f34e74f`, April 2026) is a near-exact precedent; `crypto_routes` is smaller (~450–550 LoC vs ~4,900) because it reuses unicast NLRI instead of writing a TLV parser + UI-RIB.

#### Actions taken (Prompt 2)

1. Deep-dived `bgpd/` and `lib/` for the concrete AFI/SAFI touch points (`lib/zebra.h`, `lib/iana_afi.h`, `lib/prefix.c`, `bgpd/bgpd.{c,h}`, `bgp_packet.c`, `bgp_open.c`, `bgp_attr.c`, `bgp_route.c`, `bgp_vty.c`, `bgp_debug.c`, `bgp_updgrp.h`).
2. Mined git history for the `SAFI_UNREACH` series and captured the per-commit file lists as the reference decomposition.
3. Confirmed `SAFI_MAX` bump auto-resizes ~105 `[AFI_MAX][SAFI_MAX]` arrays and that `bgp_create()`'s `FOREACH_AFI_SAFI` loop auto-allocates the new RIB.
4. Wrote the three deliverables. No FRR source modified; no build run.

---

## Prompt 3

> - Document any code changes required to complete implementing crypto_routes to FRRouting.
> - Create a validation plan to test code changes.
> - Keep adding more Prompts and Responses to `ClaudePromptAndReplies.md`.

### Response 3 — Summary

Produced two documents at the repo root:

| File | Contents |
|---|---|
| `CODE_CHANGES.md` | The definitive, **apply-ready** code-change spec — every file, every hunk, in 9-commit order, with real diff snippets verified against the current source (`bc8971168b`). Covers: `lib/` core (enum, IANA map, `safi2str`, node enum), `bgpd/bgpd.{c,h}` (`enum bgp_af_index`, `afindex()`, peer-group helpers, RIB/maxpaths), capability display, `bgp_attr.c` (4 `switch` arms + `nh_afi`), `bgp_packet.c` dispatch, `bgp_route.c` (`bgp_rd_from_dest`, FIB boundary), rfapi arms, full CLI (nodes, grammar, `bgp_node_*`, config-write, `install_element` block, `vtysh`), `show`, `debug`, docs, tests. Includes an **exhaustive `switch`/`if` site checklist** (§11) and a "deliberately NOT changed" list (§12). |
| `VALIDATION_PLAN.md` | Change-traceable validation — 10 validation gates (G0–G9), a **structured-reasoning template** to complete per change group (with a worked example for the NLRI dispatch), a **change → risk → validation → pass-criterion traceability matrix**, the detailed topotest spec (13 assertions incl. the explicit "no FIB install" negative test), negative/boundary cases, regression selection with rationale, a sign-off checklist, and a baseline-capture procedure to run before touching code. |

#### Key points established (Prompt 3)

- **18 files** change; ~450–550 LoC non-test. No new source file, no `subdir.am` / `configure.ac` change, no new dependency.
- **14 exhaustive `switch` sites** across `lib/` + `bgpd/` (+ 5 rfapi) get a `case SAFI_CRYPTO_ROUTES:` arm — `-Wswitch -Werror` makes each a build failure until done, so the compiler is the primary checklist.
- **`bgp_update()` / `bgp_nlri_parse_ip()` signatures unchanged** — verified the existing call already passes `NULL` for the trailing `unreach` params, so crypto-routes rides the unicast path untouched.
- **RIB allocation + free are automatic** — `bgp_create()` / `bgp_free()` iterate `FOREACH_AFI_SAFI`.
- **`SAFI_MAX` / `BGP_AF_MAX` bump** is the main regression risk → full `bgp_*`/`evpn_*`/`bgp_gr_*` topotest sweep + ASan + leak check are merge gates; baseline captured first.
- The FIB boundary is an **explicit test assertion** (topotest T7: `show ip route` has no crypto-routes entry), not an assumption.

#### Actions taken (Prompt 3)

1. Pulled the exact upstream diffs for the `SAFI_UNREACH` plumbing commits (`61f097d557` capability, `4154e91508` CLI + nodes + vtysh, `1a004a0281` attr/packet, `04ba8c4abb` route processing, `023da6de44` lib) and translated each hunk to `SAFI_CRYPTO_ROUTES`, dropping the TLV/UI-RIB parts crypto-routes doesn't need.
2. Verified every current `switch (safi)` / `switch (packet->safi)` site in `bgpd/` + `lib/` and the non-switch OR-chains in `bgpd.h`.
3. Confirmed the "not changed" set: zebra gates, NHT, GR-NSF, `bgp_path_info_extra`, `bgp_update()` signature.
4. Wrote `CODE_CHANGES.md` and `VALIDATION_PLAN.md`. **No FRR source modified; no build run** — that is Prompt 4 ("Go ahead with code changes").

---

## Prompt 4

> - Go ahead with code changes.
> - Validate code changes versus the Validation Plan with structured reasoning.
> - Keep adding more Prompts and Responses to `ClaudePromptAndReplies.md`.

### Response 4 — Summary

**Applied the `crypto_routes` SAFI implementation** to the FRR submodule on a new
branch `crypto-routes-safi` (off `bc8971168b`), following `CODE_CHANGES.md`, and
**validated it statically against `VALIDATION_PLAN.md`** (full compile/topotest
gates could not run on this macOS host — FRR is Linux-only, no toolchain).

#### Code changes (20 files modified, 2 new)

| Area | Files |
|---|---|
| `lib` core | `lib/zebra.h` (`SAFI_CRYPTO_ROUTES=10`, `SAFI_MAX=11`), `lib/iana_afi.h` (`IANA_SAFI_CRYPTO_ROUTES=241` + 2 map arms), `lib/prefix.c` (`safi2str`), `lib/command.h` (2 CLI nodes) |
| `bgpd` core | `bgpd.h` (`enum bgp_af_index` +2, `afindex()` ×4, 2 OR-chains), `bgpd.c` (maxpaths pin) |
| `bgpd` protocol | `bgp_open.c` (capability display ×2), `bgp_attr.c` (nexthop/prefix/size switches ×4 + `nh_afi`), `bgp_packet.c` (`bgp_nlri_parse` → `bgp_nlri_parse_ip`) |
| `bgpd` route | `bgp_route.c` (`bgp_rd_from_dest`, ext-comm JSON, `network`/`table-map` install) — **FIB gate deliberately untouched** |
| `bgpd` CLI | `bgp_vty.c` (~154 lines: nodes, grammar, `bgp_node_*`, `bgp_vty_safi_from_str`, `argv_find_and_parse_safi`, config-write, ~40 `install_element`), `bgp_vty.h` (grammar macros) |
| `bgpd` rfapi | `rfapi_import.c` ×3, `rfapi_monitor.c` ×2 (no-op `switch` arms for `--enable-bgp-vnc`) |
| `bgpd` debug | `bgp_debug.c` / `.h` (`debug bgp crypto-routes`) |
| `vtysh` | `vtysh.c` (2 nodes + `DEFUNSH` navigation + registration) |
| docs | `doc/user/crypto-routes.rst` (new), `doc/user/bgp.rst`, `doc/user/subdir.am` |
| tests | `tests/bgpd/test_capability.c` (`crypto_routes_safi_test()`, 10 asserts), `tests/topotests/bgp_crypto_routes/` (new: 3-router topo + `bgpd.conf`/`zebra.conf` ×3 + `test_bgp_crypto_routes.py`, 11 test functions) |

**Not changed** (verified): the `bgp_process_main_one()` zebra/FIB-install gate,
NHT, GR-NSF, `struct bgp_path_info_extra`, `bgp_update()`/`bgp_nlri_parse_ip()`
signatures, `bgpd/subdir.am`, `configure.ac`, `mgmtd`/`yang`.

#### Validation (`VALIDATION_RESULTS.md`)

- **G1 (compile):** static-only. All **24 `switch (safi)` arms across 10 files** +
  3 OR-chains enumerated and patched; every new symbol's declaration site
  verified; DEFUN/DEFPY help-string arity re-balanced and checked. High
  confidence it compiles (the change is almost entirely `case` labels the
  compiler would reject if missing; cross-checked against the merged
  `SAFI_UNREACH` series that did the identical 9→10 bump).
- **G9 (structured reasoning):** complete — the `VALIDATION_PLAN.md §3` template
  filled for all 9 change groups (intent / mechanism / exhaustiveness /
  invariants / failure-modes table / blast radius / test evidence).
- **G2/G3/G4/G5/G6/G7/G8:** blocked by the macOS host; tests + docs authored,
  actions carried forward to Linux CI. `EXECUTION_TESTING_PLAN.md` §2–§5 is the
  runbook.
- **4 defects found and fixed in-session** (help-string arity ×2, an unneeded
  widening the LLD called for that inspection showed was already correct, a
  missing enum trailing comma) — logged in `VALIDATION_RESULTS.md §5`.
- Highest-radius change flagged: the `lib/command.h` `enum node_type` insertion
  shifts `BFD_NODE`+ by 2 → every daemon + `vtysh` must rebuild together (same as
  `SAFI_UNREACH`); full (not just `bgp_*`) topotest suite is the real check.

#### Actions taken (Prompt 4)

1. Created branch `crypto-routes-safi` in the `frr/` submodule.
2. Applied all hunks from `CODE_CHANGES.md` commits 1–9, reading each target site
   fresh before editing.
3. Re-grepped every `switch (safi)` / `case SAFI_*` / OR-chain post-change to
   confirm completeness; reviewed the full `git diff` hunk-by-hunk.
4. Authored the unit-test additions and the topotest suite.
5. Wrote `VALIDATION_RESULTS.md` with the gate-by-gate status and the structured
   reasoning.
6. **Changes are staged on the branch but not committed** (per standing
   instruction to commit only when asked); the 9-commit breakdown in
   `CODE_CHANGES.md §13` is ready to apply on request.
