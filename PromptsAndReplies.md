# CISCO AI Assignment: FRRouting

## Prompt 1

- Perform a comprehensive analysis of this repository to build a developer foundation for future modifications.
- Identify the core programming languages, frameworks, and primary dependencies.
- Map out the directory structure and explain the purpose of key folders and entry points.
- Explain the data flow and how the main components interact with each other.
- Detail the prerequisites and exact steps needed to spin up the local development environment.
- Outline the existing testing suites and how to run them.
- Highlight any specific coding standards, patterns, or architecture rules enforced in this codebase.
- Write all of the above sections, along with a summary of the prompts used and actions taken during this session, into `ClaudePromptsAndReplies.md`.
- Ensure the output is cleanly formatted in Markdown with proper headers, code blocks, and bullet points.
- Save all your findings, structural insights, and a log of this analysis into a new file named `REPO_ANALYSIS_LOG.md`.

## Prompt 2

- From Analysis, Build a High Level Design Document v1 (in markdown) & a Low Level Design Document v1 (in markdown)
- Build a reference execution & testing plan for our Problem Statement.
- Showcase changes in HLDv1 and LLDv1 for our Problem Statement.

## Prompt 3

- Introduce a new BGP address family (AFI/SAFI) named crypto_routes.
- Document any code changes required.
- Changes to be reviewed before compilation.

---

## Session Log

### Prompt 1 — deliverables

- `REPO_ANALYSIS_LOG.md` — full codebase analysis (languages, deps, structure, data flow, dev env, tests, coding standards) + analysis log.
- `ClaudePromptsAndReplies.md` — same analysis, prefixed with the session prompt/action log.

### Prompt 2 — deliverables

- `HLD_v1.md` — High Level Design v1 for the `crypto_routes` SAFI: scope, assumed semantics, design principles, component/data-flow view, (AFI,SAFI) model, NFRs, alternatives, risks, phasing, acceptance criteria. Section 9 is the HLD-level **design-delta showcase** (before/after, change footprint, reference commit decomposition).
- `LLD_v1.md` — Low Level Design v1: file-by-file / function / enum / switch-arm change map with diff-style snippets, "what is NOT changed and why", build/codegen notes, a **pre-compilation review checklist**, a full `switch (safi)` site inventory (Appendix A), and a "if a new AFI is truly required" appendix (Appendix B).
- `EXECUTION_TESTING_PLAN.md` — reference execution & testing plan: env setup, build (with the `-Wswitch` checklist loop), static gates, unit tests to add, a full topotest design (`tests/topotests/bgp_crypto_routes/` topology + 11 assertions), regression sweep, ASan/leak run, manual smoke, CI expectations, acceptance matrix, rollback, and a 7-day timeline.

#### Key design decisions (Prompt 2)

- **New SAFI, not new AFI.** `SAFI_CRYPTO_ROUTES = 10` (`SAFI_MAX` 10→11) under existing `AFI_IP`/`AFI_IP6`. A new AFI would resize `[AFI_MAX]` arrays across all of `lib/` and every daemon for no benefit — crypto routes are IP prefixes.
- **Reuse unicast NLRI.** `bgp_nlri_parse()` dispatches `SAFI_CRYPTO_ROUTES` to the existing `bgp_nlri_parse_ip()`. No new wire format, no new parser, no new RIB type.
- **Control-plane only in v1.** Deliberately excluded from the `bgp_zebra` FIB-install gate — like BGP-LS / FlowSpec / `SAFI_UNREACH`.
- **IANA SAFI 241** (Private-Use range 241–254) — no squatting on an unassigned public number; configurable value deferred to v2.
- **Template:** the merged `SAFI_UNREACH` work (15-commit series, `023da6de44 … 1b2f34e74f`, April 2026) is a near-exact precedent; `crypto_routes` is smaller because it reuses unicast NLRI instead of writing a TLV parser + UI-RIB.

#### Actions taken (Prompt 2)

1. Deep-dived `bgpd/` and `lib/` for the concrete AFI/SAFI touch points: `lib/zebra.h`, `lib/iana_afi.h`, `lib/prefix.c`, `bgpd/bgpd.{c,h}` (`enum bgp_af_index`, `afindex()`, `bgp_map_afi_safi_*`, `bgp_create()` RIB alloc), `bgpd/bgp_packet.c` (`bgp_nlri_parse`), `bgpd/bgp_open.c`, `bgpd/bgp_attr.c`, `bgpd/bgp_route.c` (zebra gate), `bgpd/bgp_vty.c` (`bgp_node_type`/`bgp_node_afi`/`bgp_node_safi`/`bgp_vty_safi_from_str`/config-write/`cmd_node`), `bgpd/bgp_debug.c`, `bgpd/bgp_updgrp.h`.
2. Mined git history for the `SAFI_UNREACH` series and captured the per-commit file lists as the reference decomposition.
3. Confirmed `SAFI_MAX` bump auto-resizes ~105 `[AFI_MAX][SAFI_MAX]` arrays and that `bgp_create()`'s `FOREACH_AFI_SAFI` loop auto-allocates the new RIB.
4. Wrote the three deliverables above.
5. No FRR source modified; no build run. Implementation is Prompt 3.
