# mkdir.M1-001 — implementation notes

**Issue:** #1 — scaffold + caps.decl (target-parent write cap + TXN cap).
**Upstream doc:** `design/tooling/r49-r50-plan.md` §5.9 (paideia-os).

## What landed

- `caps.decl` — mkdir requires four caps: `KIND_PDXFS_TXN`,
  `KIND_PDXFS_FILE(write)`, `KIND_USER`, `KIND_IPC_ENDPOINT`. Every row
  documents the slot number the loader must seed and the specific rights
  the tool draws against. Declares one output schema
  (`CreatedDirRecord@0.1`) that M3-001 wires against libpdx-semantic-
  pipe.
- `design/architecture.md` — full internal spec: module boundary
  (`Mkdir` + `MkdirState`), `_start` flow, `mkdir_one(path)` create
  sequence, flag semantics table, sidecar cap slot map, paideia-as
  encoding conformance, and the explicit M1 non-goals.
- `src/mkdir_state.pdx` — `MkdirState` module: return-code constants
  (MK_OK, MK_ARGV_ERR, MK_MISSING_PATH, MK_MULTI_LEVEL_UNSUPPORTED,
  MK_TXN_OPEN_FAIL, MK_MKDIR_FAIL, MK_TXN_COMMIT_FAIL), sidecar slot
  number constants, placeholder op ordinals, SC+ ID constants, singleton
  `.bss` slots for `flag_p / flag_v / flag_dry_run / exit_code /
  err_pos_index`, and the `reset()` entry point.
- `src/mkdir.pdx` — `Mkdir` module: `_init_caps` sidecar array (four
  rows in the R20b.M4 on-wire format), the `mkdir: M1 ring3 ok` marker
  string, and `_start` (calls `MkdirState::reset`, emits the marker on
  `sys_debug_puts`, then `sys_exit(0)`). The M1-002 argv-parse wire-up
  and M1-003 mkdir_one create sequence go into this same file — the
  scaffold ships with the entry point wired but no orchestration yet.
- `tests/README.md` — points at `mkdir.M4-001` for the test matrix.
- `.plans/README.md` — layout pointer.
- `STATUS.md` — milestone rollup updated (M1-001 LANDED).

## Design decisions

- **Two-module split.** Follows libpdx-argv's `Parser` + `ParsedArgs`
  precedent. Keeps `MkdirState` testable in isolation at M4-001 while
  `_start` stays a straight-line orchestrator. Module names are
  PascalCase basename (`Mkdir` / `MkdirState`) — no directory prefix per
  the paideia-as conformance rule.
- **Sidecar row order = caps.decl row order.** The `_init_caps` array
  encodes the four rows in the same order the `caps.decl` `requires:`
  block lists them. When M2 adds a fifth row (e.g. a KIND_IPC_ENDPOINT
  audit sink), both files grow together.
- **Placeholder cap invocations at M1-003.** `mkdir_one` at M1-003 will
  use `PXT_OP_QUERY_ID` on slot 0 and `PFF_OP_DEBUG_PRINT` on slot 1 as
  stand-ins for the create-dir op that the kernel does not yet expose.
  Same shape as libpdx-cap M1's `cap_manifest_verify` skeleton — the
  signature and call sites are frozen at M1 so the M2 patch is a body
  edit rather than a signature change. Filed as `mkdir.M2-substrate-001`
  at the parent repo for the kernel-side follow-up.
- **`sys_debug_puts` as the M1 stderr proxy.** M3-001 wires the real
  KIND_IPC_ENDPOINT at slot 3 with a semantic-pipe schema; until then
  every diagnostic goes through `sys_debug_puts` (SC+ ID 12). Same
  choice libpdx-argv would make if it emitted diagnostics.
- **Inline SC+ IDs, no syscall_shim link.** Follows the echo_client.pdx
  discipline — the tool binary stays self-contained and does not pull
  in the syscall_shim.pdx object file. IDs used at M1-001: 12 and 60.
  IDs added by M1-003: 4 (`sys_cap_invoke`).

## paideia-as conformance

- Module names PascalCase basename (`Mkdir`, `MkdirState`) with no
  directory prefix.
- No `test` mnemonic anywhere; every zero-check is `cmp reg, 0`.
- Every `cmp reg, imm` uses an immediate ≤ 0x7FFFFFFF. Max immediate
  seen at M1-001: 60 (the sys_exit SC+ ID). M1-003 will bring 0x2F (the
  `/` byte for the multi-level guard).
- `r11` used only as scratch (LEA temps for cross-module references).
  Never live across a call.
- Byte loads (added at M1-002 flag walk and M1-003 path walk) use
  `xor rax, rax; mov_b rax, [ptr]` per the paideia-as #1248 mitigation
  pattern.
- Labels avoid every paideia-as reserved keyword (`loop`, `if`, `let`,
  `fn`, `pub`, `mut`, `struct`, `structure`, `unsafe`, `block`). Every
  label in `src/mkdir.pdx` uses the `mkdir_` prefix; every label in
  `src/mkdir_state.pdx` (added later) uses the `mks_` prefix. Recorded
  in `design/architecture.md` §7 as a persistent gotcha per the
  paideia-as reserved-labels memory note.

## Cross-module linkage

`src/mkdir.pdx` references `MkdirState::reset` by qualified name. It
does not yet touch any `flag_*` slot — those come at M1-002. The
libpdx-argv `Parser::parse_argv` reference lands at M1-002.

## What did not land (queued for M1-002 and beyond)

- Argv parse + flag walk (`parse_flags_from_argv`) — M1-002.
- `mkdir_one(path)` single-level create + TXN scaffold — M1-003.
- `mkdir_run()` positional walk — M1-003.
- `emit_stderr(buf, len)` helper — M1-002 (added when the first
  diagnostic emit site lands).
- Multi-level `-p` create sequence — M2-001.
- Pre-existing-dir idempotence under `-p` — M2-002.
- Cap-tail owner write — M2-003.
- `CreatedDirRecord[]` emission on the semantic pipe — M3-001.
- `CreateDirRecord` audit journal entry — M3-002.
- PdxFS v1 undo record + `undo mkdir` — M3-003.
- Coreutil test matrix — M4-001.

## Build note

mkdir M1 has no local build script yet. paideia-as ≥ v0.33 (for the
`mov_b` narrow-load mnemonic + the `@align` attribute) will build both
modules once main invokes
`paideia-as build src/mkdir_state.pdx src/mkdir.pdx -o build/mkdir.elf`
— the exact invocation is a mkdir.M2 concern, not M1.

Build is main-only per the no-background-builds memory note; this issue
lands the source and defers the build check to main's synchronous
`bash tools/build.sh` step.
