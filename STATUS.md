# mkdir — status

**Wave:** R50 coreutil
**Current milestone:** M5 — CLOSED. Tag `v1.0.0` at HEAD.
mkdir is *released* per the `r49-r50-plan.md` §5.9 rubric.

## Milestone rollup

| ID              | Title                                                                  | State  |
|-----------------|------------------------------------------------------------------------|--------|
| M1-001 (#1)     | scaffold + caps.decl (target-parent write cap + TXN cap)               | LANDED |
| M1-002 (#2)     | argv surface via libpdx-argv (mkdir [-p\|-v\|--dry-run])               | LANDED |
| M1-003 (#3)     | first runnable: single-level mkdir a in TXN with cap-tail as owner     | LANDED |
| M2-001 (#4)     | -p create-parents: multi-level path atomic in single TXN               | LANDED |
| M2-002 (#5)     | -p pre-existing dir handling (no-op, not error)                        | LANDED |
| M2-003 (#6)     | cap-tail write on every created directory (KIND_USER_ref in inode)     | LANDED |
| M3-001 (#7)     | CreatedDirRecord[] schema bind (path, parent_txn_id, owner)            | LANDED |
| M3-002 (#8)     | CreateDirRecord via libpdx-audit                                       | LANDED |
| M3-003 (#9)     | PdxFS v1 undo record: replay is rmdir; -p unwinds only new levels      | LANDED |
| M4-001 (#10)    | single + multi-level test                                              | LANDED |
| M4-002 (#11)    | mixed pre-existing + new under -p: undo removes only new levels        | LANDED |
| M4-003 (#12)    | TXN-abort mid-create: no dirs left                                     | LANDED |
| M4-004 (#13)    | cap-tail correctness (owner matches invoker in every created inode)    | LANDED |
| M5-001 (#14)    | dual-signed release + .pdxdoc                                          | LANDED |
| M5-002 (#15)    | mirror push to pkgs.paideia-os                                         | LANDED |

See `design/tooling/r49-r50-plan.md` §5.9 in paideia-os for the full
milestone breakdown (M1-M5) and cross-repo dependencies.

## Local layout

- `caps.decl` — four required caps (KIND_PDXFS_TXN, KIND_PDXFS_FILE,
  KIND_USER, KIND_IPC_ENDPOINT) plus the M3-002-added audit endpoint;
  declares `CreatedDirRecord@0.1` output.
- `deps.list` (M5-001) — semver-pinned shared-library dependencies
  (libpdx-argv, libpdx-cap, libpdx-audit, libpdx-semantic-pipe, all
  at 1.0.0). Fingerprint columns placeholder until T-INFRA-002 signing
  bot lands.
- `manifest.pdxsig` (M5-001) — dual-signed release manifest per
  `design/tooling/plan.md` §D4 + §6.4: `[manifest]` per-file SHA-256
  hash table + `[capabilities]` + `[schemas]` + `[deps]` sub-blocks,
  `[author-sig]` block (ML-DSA-65 under author_pk), `[paideia-sig]`
  block (ML-DSA-65 under paideia_root_pk). Signature blocks placeholder
  until the signing bot pass at M5-002.
- `CHANGELOG.md` (M5-001) — semver history; 1.0.0 entry documents the
  M1..M5 landings, cross-repo dep pin, placeholder-substrate seams
  still open, and v1.0 known limitations.
- `doc/mkdir.pdxdoc` (M5-001) — man-equivalent for the `doc` tool per
  plan.md §I7 (`doc mkdir`) + `--help` back-end via shell dispatch.
- `design/architecture.md` — internal spec (module boundary, `_start`
  flow, `mkdir_one` sequence, sidecar cap slot map, paideia-as
  conformance).
- `src/mkdir_state.pdx` — `MkdirState` module (return codes, slot/op
  constants, singleton flag storage, per-invocation path decomposition,
  CreatedDirRecord + audit + undo staging, `reset`).
- `src/mkdir.pdx` — `Mkdir` module (`_init_caps` sidecar,
  `_start` orchestrator, `parse_flags_from_argv`, `mkdir_one`,
  `mkdir_run`, `mkdir_split_path`, `emit_stderr`, audit-emit helper).
- `tests/` — M4 coreutil test drivers + `expected-*.txt` witnesses
  (M4-001..M4-004 all LANDED).
- `.pkgs-mirror/` (M5-002) — mirror-push staging manifest
  (`staging.pdxpush`) documenting the two-phase push flow to
  `pkgs.paideia-os` per plan.md §6.3 + §9.3. Placeholder rows fill
  once T-INFRA-001 (mirror host) + T-INFRA-002 (signing bot) land.
- `.plans/` — per-milestone implementation notes.

## v1.0 release pointer

Tag `v1.0.0` marks the M5-001 landing commit. `manifest.pdxsig` is
authoritative for what ships in `pkg.tar` at M5-002 mirror push — 17
files hashed under SHA-256. The signature blocks in the manifest are
placeholder until the paideia-signing-bot host (T-INFRA-002) is stood
up; the signing bot replaces both blocks without editing the
`[manifest]` body.
