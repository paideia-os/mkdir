# mkdir — status

**Wave:** R50 coreutil → v1.1-A extraction
**Current milestone:** v1.1-A — LANDED in the Unreleased section of
`CHANGELOG.md`. Retires the M1-001 STUB and wires the real
`sys_mkdir` syscall (paideia-os sysno 79, kernel body at
`src/kernel/core/syscall/sys_mkdir.pdx`, R56.M3-004 #1793).

`mkdir` at HEAD is the minimum viable body a paideia-os user can
invoke and observe a directory afterwards. Every M1..M5 seam that
was placeholder-only (cap-invoke stubs, static bootstrap argv,
record-staging, audit-journal hand-roll) has been retired; see the
Unreleased v1.1-A entry in `CHANGELOG.md` for the file-by-file
inventory.

## v1.1-A behaviour

    mkdir <PATH> [<PATH>...]

- Reads argv from rdi/rsi per the frozen execve ABI (paideia-os
  `design/user/execve-abi.md`).
- Walks argv[1..argc], calling `sys_mkdir` with mode 0755 (0x1ED)
  and `path_len_hint` 255 (walker CAP per
  `design/user/syscall-table.md`) on each.
- Exit 0 on all-success; exit -errno on the first `sys_mkdir` that
  returns non-zero (verbatim through `sys_exit`'s status word).
- Exit 2 with `mkdir: missing operand\n` on the debug channel when
  `argc < 2`.
- No flags recognised at v1.1-A: no `-p`, no `-m`, no `-v`, no
  `--dry-run`, no `--schema`, no `--version`. `--help` remains
  shell-dispatched to `doc mkdir`.

## Local layout

- `caps.decl` — `requires: []` and `declares_output_schemas: []`;
  v1.1-A takes no caps at exec (`sys_mkdir` is a plain @{fs}
  syscall, not cap-gated).
- `deps.list` — empty at v1.1-A; the M5-era pins on `libpdx-argv`,
  `libpdx-cap`, `libpdx-audit`, `libpdx-semantic-pipe` are all
  retired.
- `manifest.pdxsig` — carries the M5-era body verbatim (signed
  block frozen at v1.0 per `design/tooling/plan.md` §D4 in
  paideia-os; the T-INFRA-002 signing bot pass will replace it
  when v1.1-A cuts a `1.1.0` tag).
- `CHANGELOG.md` — v1.1-A entry lives in the Unreleased section;
  cuts to `1.1.0` when tagged.
- `doc/mkdir.pdxdoc` — carries the M5-era text; a doc-truth pass
  paired with the v1.1-A extraction is filed as a v1.1-A follow-up
  (SYNOPSIS drops the retired flags; FLAGS narrows to the
  positional operand; EXIT CODES documents the `0 / 2 / -errno`
  triad).
- `design/architecture.md` — v1.1-A section supersedes §§2..8 of
  the M1..M5 body.
- `design/enhancement-plan.md` — M5-era enhancement plan; every
  open item at that plan's tail is absorbed (retired or deferred)
  by the v1.1-A extraction.
- `src/mkdir.pdx` — v1.1-A body: one straight-line `_start`
  reading argv, looping `sys_mkdir` per positional.
- `src/mkdir_state.pdx` — DELETED at v1.1-A.
- `tests/` — M4 + ENH-012 drivers and witnesses all deleted at
  v1.1-A (see `tests/README.md` for the v1.1-A smoke-witness
  follow-ups).
- `.pkgs-mirror/` — mirror-push staging manifest untouched; the
  T-INFRA-001 mirror host + T-INFRA-002 signing bot follow-up
  bounds when a v1.1-A dual-signed manifest.pdxsig can ship.
- `.plans/` — per-milestone implementation notes; the v1.1-A
  extraction lands one further note (`v1.1-A-notes.md`, filed
  as a follow-up).

## v1.2-A follow-ups (deferred from v1.1-A)

- `-p` multi-level with parent creation, rebuilt on top of the
  real `sys_mkdir` primitive (walker becomes a straight per-
  component syscall with `sys_stat` for pre-existing hits).
- `-m <mode>` custom mode parsing.
- Audit-journal emit on top of the real `libpdx-audit`
  marshaller.
- `CreatedDirRecord@0.1` semantic-pipe emit on top of a real
  `libpdx-semantic-pipe send_record` consumer.

## Milestone rollup

| ID              | Title                                                                  | State  |
|-----------------|------------------------------------------------------------------------|--------|
| v1.1-A          | Real body extraction: retire M1-001 STUB, wire sys_mkdir               | LANDED |
| ENH-004..012 (#) | Enhancement v1.x -- absorbed by v1.1-A (retired or deferred to v1.2-A) | LANDED |
| M1..M5 (#1..#15) | Placeholder scaffold (v1.0.0 tag)                                     | LANDED |

See `CHANGELOG.md` for the per-milestone landing detail.
