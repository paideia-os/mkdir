# mkdir.M1-002 — implementation notes

**Issue:** #2 — argv surface via libpdx-argv (`mkdir [-p|-v|--dry-run]`).
**Upstream doc:** `design/tooling/r49-r50-plan.md` §5.9 (paideia-os).

## What landed

- **Symbol rename** in `src/mkdir_state.pdx`: `MkdirState::reset` →
  `mkdir_state_reset`. paideia-as resolves cross-module calls by
  unqualified linker name (no `Module::name` qualifier is legal in the
  assembler surface), so a bare `reset` in mkdir_state.pdx would collide
  with libpdx-argv's own `reset` symbol in `parsed_args.pdx:33` the
  moment mkdir starts consuming libpdx-argv. Verified via
  `grep -rn "call [A-Z][a-zA-Z]*::" paideia-os/src` — zero hits.
- `src/mkdir.pdx` extended with the M1-002 wire-up:
  - Bootstrap argv static strings: `argv_flag_v` (`"-v\0"`),
    `argv_flag_dry_run` (`"--dry-run\0"`), `argv_positional_a` (`"a\0"`).
  - `argv_bootstrap_ptrs : [u64; 3]` (mut, built at `_start` entry via
    three `lea + mov` stores) + `argv_bootstrap_argc = 3`.
  - Stderr strings: `err_argv_msg` (24 bytes), `err_no_operand_msg`
    (23 bytes).
  - Well-known flag byte tables: `flag_p_name` (`"p\0"`),
    `flag_v_name` (`"v\0"`), `flag_dry_run_name` (`"dry-run\0"`) —
    documentation-only for M1; the recognizer inlines the byte constants
    directly per libpdx-argv precedent.
  - `emit_stderr(buf, len)` — thin `sys_debug_puts` wrapper kept as its
    own symbol so mkdir.M3-001's rewrite is a one-line body swap.
  - `parse_flags_from_argv()` — walks
    `ParsedArgs::flag_names[0..flag_count]`, byte-compares each entry
    against `p` / `v` / `dry-run`, sets `MkdirState::flag_p /
    flag_v / flag_dry_run` to 1 on match.
  - Updated `_start` orchestrator (see next section).
- `.plans/m1-002-notes.md` (this file).
- STATUS.md — M1-002 rollup bumped to LANDED.

## `_start` M1-002 flow

```
Step 0  ring3 marker on sys_debug_puts
Step 1  call mkdir_state_reset
Step 2  call reset                        // libpdx-argv ParsedArgs::reset
Step 3  build argv_bootstrap_ptrs         // three lea+mov stores
Step 4  call parse_argv(argv, argc)
          rax != 0 → emit err_argv_msg + sys_exit(MK_ARGV_ERR)
Step 5  call parse_flags_from_argv
Step 6  pos_count == 0 → emit err_no_operand_msg + sys_exit(MK_MISSING_PATH)
Step 7  sys_exit(MK_OK)                   // mkdir_run wire-up at M1-003
```

The bootstrap argv (`-v --dry-run a`) exercises three shapes end to end:
one short flag, one long flag, and one positional. The `-p` short flag
is not in the bootstrap but is still exercised by
`parse_flags_from_argv` — the recognizer is data-driven and the code
path that matches `p\0` fires only if the caller supplies such a name,
so the test doesn't exercise the `-p` write branch; the write branch is
covered by the M4-001 test matrix.

## Design decisions

- **Bootstrap argv over shell delivery.** Shell.M4 will publish a
  stable argv-in-sidecar shape; until then _start builds its argv
  statically. This mirrors echo_client's hardwired `"PAIDEIA!"` payload
  — the point at M1 is to exercise the parse + flag walk with a
  deterministic input, not to plumb the shell.
- **Well-known flag inline compare.** libpdx-argv predates any strcmp
  helper (see its own pdx-schema inline compare at
  `parser.pdx:151..161`). We repeat the pattern rather than factor a
  helper; the M2 `well_known_flag_id(name) → i64` in libpdx-argv will
  absorb every consumer's table.
- **Silent-ignore of unknown flags at M1-002.** libpdx-argv.M2-002 will
  land the 9-flag standard-vocabulary rejection path; until then a
  stray `--foo` on the mkdir CLI is stored in `flag_names` but never
  matched by the recognizer, so it's a no-op. Explicit rejection is a
  libpdx-argv concern, not a mkdir concern.
- **Diagnostic + exit code, no partial run.** Argv-error and
  missing-operand both terminate `_start` via `sys_exit` before any
  `mkdir_one` runs. There is no "some directories created, some failed"
  state at M1 because there is no create loop yet.

## paideia-as conformance

- Module names PascalCase basename (`Mkdir`, `MkdirState`). No
  directory prefix.
- No `test` mnemonic; every zero-check uses `cmp reg, 0` or `cmp
  reg, imm`.
- Every `cmp reg, imm` uses an immediate ≤ 0x7FFFFFFF. Max seen at
  M1-002: `0x7A` (`'z'` upper bound in the ASCII compares — actual
  compares only reach `0x79`).
- `r11` used only as scratch (LEA temps in `parse_flags_from_argv` and
  in the `_start` argv bootstrap build). Never live across a call.
- Byte loads (both in `parse_flags_from_argv` and in the flag-name
  table match) use `xor rax, rax; mov_b rax, [ptr]` per the paideia-as
  #1248 mitigation.
- Labels avoid every paideia-as reserved keyword. Prefixes: `mkdir_` in
  `_start`, `mkdir_pff_` in `parse_flags_from_argv`, `mks_` (reserved
  but unused at M1-002 — the state module has no labels yet).
- Every helper (`emit_stderr`, `parse_flags_from_argv`,
  `mkdir_state_reset`) is a leaf function — no push/pop parity to
  preserve.

## Cross-module linkage

- `src/mkdir.pdx` calls the following symbols by unqualified linker
  name:
  - `mkdir_state_reset` → local (`src/mkdir_state.pdx`).
  - `reset` → libpdx-argv (`src/parsed_args.pdx`, `ParsedArgs::reset`).
  - `parse_argv` → libpdx-argv (`src/parser.pdx`, `Parser::parse_argv`).
  - `parse_flags_from_argv`, `emit_stderr` → local.
- `src/mkdir.pdx` reads the following `.bss` symbols by unqualified
  linker name (paideia-as' cross-module data pattern per
  `src/user/dispatch.pdx` → `Tokenizer::argc`):
  - `flag_names`, `pos_count` → libpdx-argv `ParsedArgs`.
  - `flag_p`, `flag_v`, `flag_dry_run` → local `MkdirState`.

## What did not land (queued for M1-003 and beyond)

- `mkdir_one(path)` create sequence (single-level, TXN-scoped) —
  M1-003.
- `mkdir_run()` positional walk — M1-003.
- `-p` recognition write branch exercised (bootstrap argv lacks `-p`) —
  M4-001 matrix.
- Unknown-flag rejection — libpdx-argv.M2-002.
- Real argv-in-sidecar wiring — mkdir.M2 (deps: shell.M4).
- Emit `CreatedDirRecord[]` on the semantic pipe — M3-001.

## Build note

Same as M1-001: mkdir has no local build script yet. paideia-as ≥ v0.33
will build both modules once main invokes
`paideia-as build src/mkdir_state.pdx src/mkdir.pdx -o build/mkdir.elf`
alongside a libpdx-argv link. Build is main-only per the
no-background-builds memory note.
