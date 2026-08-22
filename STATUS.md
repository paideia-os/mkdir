# mkdir — status

**Wave:** R50 coreutil
**Current milestone:** M2 (core implementation) — IN PROGRESS. M1
(design + skeleton) CLOSED.

## Milestone rollup

| ID              | Title                                                                  | State  |
|-----------------|------------------------------------------------------------------------|--------|
| M1-001 (#1)     | scaffold + caps.decl (target-parent write cap + TXN cap)               | LANDED |
| M1-002 (#2)     | argv surface via libpdx-argv (mkdir [-p\|-v\|--dry-run])               | LANDED |
| M1-003 (#3)     | first runnable: single-level mkdir a in TXN with cap-tail as owner     | LANDED |
| M2-001 (#4)     | -p create-parents: multi-level path atomic in single TXN               | LANDED |
| M2-002 (#5)     | -p pre-existing dir handling (no-op, not error)                        | OPEN   |
| M2-003 (#6)     | cap-tail write on every created directory (KIND_USER_ref in inode)     | OPEN   |

See `design/tooling/r49-r50-plan.md` §5.9 in paideia-os for the full
milestone breakdown (M1-M5) and cross-repo dependencies.

## Local layout

- `caps.decl` — four required caps (KIND_PDXFS_TXN, KIND_PDXFS_FILE,
  KIND_USER, KIND_IPC_ENDPOINT); declares `CreatedDirRecord@0.1` output.
- `design/architecture.md` — internal spec (module boundary, `_start`
  flow, `mkdir_one` sequence, sidecar cap slot map, paideia-as
  conformance, M1 non-goals).
- `src/mkdir_state.pdx` — `MkdirState` module (return codes, slot/op
  constants, singleton flag storage, `reset`).
- `src/mkdir.pdx` — `Mkdir` module (`_init_caps` sidecar, `_start`
  orchestrator, `parse_flags_from_argv`, `mkdir_one`, `mkdir_run`,
  `emit_stderr`).
- `tests/` — empty until `mkdir.M4-001` lands the coreutil test matrix.
- `.plans/` — per-milestone implementation notes.
