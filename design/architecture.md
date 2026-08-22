# mkdir — architecture

**Wave:** R50 coreutil
**Repo:** github.com/paideia-os/mkdir
**Upstream design:** `design/tooling/r49-r50-plan.md` §4.9 + §5.9 in
[paideia-os](https://github.com/paideia-os/paideia-os).

This document describes the internal shape of the `mkdir` tool. It does
not repeat the wave-level rationale from the paideia-os plan doc; read
that first for the D3 flag-grammar contract, the D4 signed-cap-tail
requirement, and the I5 undo obligation that mkdir inverts as `rmdir`
under `-p`-created levels.

## 1. Public surface

mkdir has one entry point (`_start`) and one CLI shape:

```
mkdir [-p] [-v] [--dry-run] <path> [<path>...]
```

The invocation contract from the shell:

- argv arrives via the loader's argv/argc convention (M1 sources it
  through the shell's `Tokenizer::argv_buf` symbol per the paideia-as
  cross-module linker-name pattern that `src/user/dispatch.pdx` uses;
  M2 replaces this with a stable argv-in-sidecar shape once shell.M4
  publishes it).
- Four caps arrive via the loader's InitCap sidecar in the slots named
  by `caps.decl` (KIND_PDXFS_TXN, KIND_PDXFS_FILE, KIND_USER,
  KIND_IPC_ENDPOINT). The `_init_caps` array in `src/mkdir.pdx`
  encodes the same four rows in the sidecar's on-wire format.
- Exit code 0 on success; a `MkdirState::MK_*` return code otherwise
  (all documented in `src/mkdir_state.pdx`).

The tool never links `pkg` or `shell` at build time. Its dependencies
at M1 are strictly:

- **libpdx-argv M1-002** for `Parser::parse_argv` + `ParsedArgs`.
- **libpdx-cap M1-001** for `Cap` + `cap_manifest_verify` (skeleton at
  M1; the verify becomes strict at libpdx-cap M2-001 without a signature
  change at this end).

No dependency on libpdx-audit or libpdx-semantic-pipe at M1 — those
land at mkdir.M3.

## 2. Module boundary

Two `.pdx` modules ship at M1, matching the two-module split in
libpdx-argv (`Parser` + `ParsedArgs`):

- `MkdirState` (`src/mkdir_state.pdx`) — return-code constants, sidecar
  slot number constants, singleton `.bss` slots for the parsed flags
  (`flag_p`, `flag_v`, `flag_dry_run`), the last error's positional
  index, and a `reset()` entry point.
- `Mkdir` (`src/mkdir.pdx`) — `_init_caps` sidecar declaration, static
  argv storage (see §3.4), the four helper procedures
  (`parse_flags_from_argv`, `emit_stderr`, `mkdir_one`, `mkdir_run`),
  and `_start`.

The split follows the same rationale as libpdx-argv: keep the state
declaration testable in isolation (a future MkdirState round-trip test
lands at M4-001 alongside the coreutil test matrix) while the entry
point stays a straight-line orchestrator.

## 3. `_start` shape

`_start` is a one-shot flow that ends in `sys_exit` (never returns).
The M1 sequence is:

```
Step 0  MkdirState::reset()           — clear flag_* / exit_code / err_pos
Step 1  ParsedArgs::reset()           — clear libpdx-argv singletons
Step 2  Parser::parse_argv(argv, argc) — libpdx-argv M1-002 grammar
Step 3  if ParsedArgs::error_code != ERR_OK
          emit_stderr("mkdir: argv parse error\n")
          sys_exit(MK_ARGV_ERR)
Step 4  parse_flags_from_argv()       — walk flag_names, set flag_p/v/dry
Step 5  if ParsedArgs::pos_count == 0
          emit_stderr("mkdir: missing operand\n")
          sys_exit(MK_MISSING_PATH)
Step 6  mkdir_run()                   — foreach positional: mkdir_one(pos)
Step 7  sys_exit(exit_code)
```

`mkdir_run()` walks `pos_ptrs[0..pos_count]` and calls
`mkdir_one(path)` on each; on the first non-OK return it stores the
error code into `exit_code` and stops the loop. `mkdir_one` (§4) is the
single-directory create sequence.

## 4. `mkdir_one(path)` — create sequence (M1 baseline + M2-001 -p walk)

```
rdi = path (NUL-terminated string)
rax = MK_OK on success or one of MK_TXN_OPEN_FAIL / MK_MKDIR_FAIL /
      MK_TXN_COMMIT_FAIL / MK_MULTI_LEVEL_UNSUPPORTED
```

The M1 body:

1. **Multi-level guard.** Walk `path` looking for `'/'` (0x2F). Any hit
   returns `MK_MULTI_LEVEL_UNSUPPORTED` at M1 — REGARDLESS of `flag_p`
   (M1 accepts single-level only). `-p` is stored and recognised by the
   flag walker at M1-002 but has no code branch until `mkdir.M2-001`
   lands the multi-level TXN. The M1 code path is intentionally
   permissive of the flag on the argv and strict on the path, so an M2
   patch does not have to rewrite the CLI grammar to enable it.
2. **-v diagnostic** (if `flag_v == 1`). DEFERRED at M1-003: the
   verbose line lands with the `CreatedDirRecord` text-render at
   `mkdir.M3-001`. Emitting the path here at M1 needs a `strlen`
   helper `sys_debug_puts` demands a byte-count; the M3 semantic-pipe
   send_record carries a typed record so `strlen` is not needed on the
   `-v` path once M3 lands. `flag_v` is stored at M1-002 and read at
   M3-001; M1-003 does not read it.
3. **--dry-run short-circuit** (if `flag_dry_run == 1`). Return `MK_OK`
   without touching either cap. This branch exists at M1 so downstream
   tests can exercise the argv path in isolation from the still-being-
   built pdxfs create-dir substrate.
4. **Open TXN.** `sys_cap_invoke(SLOT_TXN, PXT_OP_QUERY_ID)`. At M1 this
   is a placeholder invocation — the real "open TXN" op lands kernel-
   side alongside the create-dir op. QUERY_ID exercises the cap
   dispatch path so a mis-seeded sidecar (wrong kind at slot 0)
   surfaces as a syscall error at M1 rather than at M2. A negative
   rax → return `MK_TXN_OPEN_FAIL`.
5. **Create dir.** `sys_cap_invoke(SLOT_PARENT_FILE, PFF_OP_DEBUG_PRINT)`.
   Again a placeholder — the real create-dir op (with the path string
   marshalled through the sidecar or an IPC endpoint) lands at M2. The
   placeholder pins the cap dispatch through KIND_PDXFS_FILE so a
   wrong-kind slot 1 is caught at M1. A negative rax → return
   `MK_MKDIR_FAIL`.
6. **Cap-tail owner** — deferred to M2-003. At M1 the invoker's user cap
   sits in slot 2 unused by `mkdir_one`; the sidecar declaration is the
   commitment that M2 draws against.
7. **Commit TXN.** `sys_cap_invoke(SLOT_TXN, PXT_OP_QUERY_STATE)` —
   placeholder for TXN commit; kernel-side commit op lands at M2. A
   negative rax → return `MK_TXN_COMMIT_FAIL`.
8. Return `MK_OK`.

The placeholder pattern (steps 4/5/7) is the exact idiom libpdx-cap's
M1 `cap_manifest_verify` uses — the signature and call sites are frozen
at M1 so the M2 patch is a body edit rather than a signature change.
Kernel-side, `KIND_PDXFS_TXN` and `KIND_PDXFS_FILE` expose only
`PXT_OP_QUERY_*` and `PFF_OP_QUERY_*` + `DEBUG_PRINT` at HEAD
(`src/kernel/core/cap/kind_pdxfs_{txn,file}.pdx`, R48b substrate-prep
#1623/#1624); the create-dir op is a `mkdir.M2-substrate` follow-up
tracked in the paideia-os repo.

## 4a. `mkdir_split_path(path)` — M2-001 component splitter

Signature: `mkdir_split_path : (u64) -> u64 !{mem} @{}` (leaf).

`rdi` in = path; `rax` out ∈ { `MK_OK`, `MK_ABS_PATH_UNSUPPORTED`,
`MK_PATH_TOO_DEEP` }.

Populates three `MkdirState` `.bss` slots:

- `comp_start_offsets[i]` — byte offset within `path` where component
  `i` begins.
- `comp_lengths[i]` — byte length of component `i` (excludes the
  trailing `'/'` or `NUL`).
- `comp_count` — number of non-empty components (0..16).

Rules:

- Path starting with `'/'` → `MK_ABS_PATH_UNSUPPORTED` immediately
  (v1.0 invoker-subtree constraint per `r49-r50-plan.md` §5.9).
- Empty components (from `foo//bar` or trailing `/`) are silently
  skipped — no error, no entry recorded.
- Empty path (first byte `NUL`) → `MK_OK` with `comp_count = 0`
  (POSIX-idempotent `mkdir -p ""`).
- Dot components (`foo/./bar`) are NOT normalised at M2 — dot
  propagates to the placeholder create as a level named `.`.
  Dot-elimination is a kernel-side concern deferred to
  `mkdir.M2-substrate-001`.
- More than `PATH_MAX_COMPONENTS = 16` non-empty levels →
  `MK_PATH_TOO_DEEP`, checked at every close rather than only at end
  so overflow surfaces at the first over-limit level.

Only `comp_count` needs zeroing at `mkdir_state_reset` — the two arrays
are consumed by index up to `comp_count`, so trailing entries are
unreachable (same discipline `ParsedArgs::reset` uses for
`flag_names`/`pos_ptrs`).

## 4b. `mkdir_one` M2-001 additions

The M1 body (§4 above) has three additions at M2-001:

1. **Step 1 gated on `flag_p`.** Under `-p`, the multi-level `'/'` guard
   is bypassed and `mkdir_split_path` handles the path decomposition.
   Under no `-p`, the guard still fires (POSIX-canonical); the caller
   then sets `comp_count = 1` so the walk-loop shape is unified.
2. **New Step 6: per-component walk.** `MkdirState::walk_i` is the
   `.bss` loop counter that survives the placeholder `sys_cap_invoke`
   without a callee-save push (same `.bss`-over-`rbx` idiom
   `mkdir_run` already uses). Each iteration calls the placeholder
   create + increments `newly_created_count`. `newly_created_count` is
   the counter the M3-003 undo record reads to unwind exactly the
   levels this invocation created.
3. **Split validation runs BEFORE the `--dry-run` short-circuit.**
   `mkdir -p --dry-run /abs/path` still surfaces
   `MK_ABS_PATH_UNSUPPORTED` — POSIX-canonical validate-not-execute.

Every non-`MK_OK` return under M2-001 still routes through
`mkdir_one_epilogue`, preserving the one-push / one-pop `rbx` parity
from M1.

## 5. Flag semantics at M1

| flag        | M1 behaviour                                             | M2 wire-up                    |
|-------------|----------------------------------------------------------|-------------------------------|
| `-p`        | LANDED at M2-001. `mkdir_one` bypasses the multi-level   | Pre-existing dir handling     |
|             | guard when `flag_p == 1` and routes the path through     | is M2-002; cap-tail stamp is  |
|             | `mkdir_split_path`. Walk iterates over `comp_count`      | M2-003.                       |
|             | components in a single TXN scope.                        |                               |
| `-v`        | Detected + stored. `mkdir_one` emits a stderr line per   | Format upgraded to `CreatedDirRecord` |
|             | created directory.                                       | text-render at M3-001.        |
| `--dry-run` | Detected + stored. `mkdir_one` returns `MK_OK` before    | Unchanged; extended at M4-004 |
|             | touching either cap.                                     | cap-tail correctness test.    |
| positional  | `ParsedArgs::pos_ptrs[0..pos_count]`. Every positional   | Unchanged.                    |
|             | is a candidate path; empty argv → `MK_MISSING_PATH`.     |                               |

The flag walk is a byte-exact compare per name in
`ParsedArgs::flag_names`, mirroring libpdx-argv's own well-known
`--pdx-schema` compare. Well-known-flag lookup will be factored into a
libpdx-argv M2 helper (`well_known_flag_id(name) → i64`) when the 9-flag
standard vocabulary lands; until then every consumer inlines its own
byte-compare chain.

## 6. Sidecar cap slot map

Four rows in `_init_caps` (see `caps.decl` for the full descriptions):

| slot | kind                 | rights                                     |
|------|----------------------|--------------------------------------------|
|  0   | `KIND_PDXFS_TXN`     | `R_PDXFS_TXN_INVOKE \| R_PDXFS_TXN_OBSERVE`|
|  1   | `KIND_PDXFS_FILE`    | `R_PDXFS_FILE_INVOKE`                      |
|  2   | `KIND_USER`          | `R_USER_INVOKE`                            |
|  3   | `KIND_IPC_ENDPOINT`  | `R_IPC_WRITE \| R_IPC_INVOKE`              |

The slot map is stable across M1–M5. Additional caps (e.g. a second
`KIND_IPC_ENDPOINT` for the audit sink at M3-002) go into slots 4+; the
declaration order in `caps.decl` matches the sidecar row order.

## 7. Compliance with paideia-as encoding constraints

Both modules follow the constraints called out in
`design/kernel/paideia-as-conformance.md` (paideia-os repo) as they
apply to userspace tooling at v0.33+:

- Module names are PascalCase basename (`Mkdir`, `MkdirState`) — no
  directory prefix.
- No `test` mnemonic; every zero-check is `cmp reg, 0`.
- Every `cmp reg, imm` uses an immediate ≤ 0x7FFFFFFF (max seen: `0x7F`
  for ASCII compare; SC+ IDs 4/12/60 all ≤ 60).
- Register `r11` is scratch, used only inside a straight-line body and
  never assumed live across a call.
- Byte loads use `xor rax, rax; mov_b rax, [ptr]` per the paideia-as
  #1248 mitigation pattern.
- Every non-leaf helper preserves SysV push/pop parity: `mkdir_one`,
  `mkdir_run`, `parse_flags_from_argv`, and `emit_stderr` all balance
  their prologue and epilogue push counts.
- Labels never use paideia-as reserved keywords (`loop`, `if`, `let`,
  `fn`, `pub`, `mut`, `struct`, `structure`, `unsafe`, `block`); every
  label in the module uses the `mkdir_` or `mks_` prefix. Recorded here
  as a persistent gotcha per the paideia-as reserved-labels rule.

## 8. What M2-001 explicitly does not do

Called out here so a reader of M2-001 code does not mistake absence
for bug:

- No pre-existing-dir handling. Under `-p` (M2-002), `mkdir a/b` where
  `a` already exists is a no-op on `a`, a create on `b`. M2-001 walks
  every component as if newly-created and calls the placeholder create
  per level; M2-002 slots the existence probe in without a signature
  change to `mkdir_one`.
- No cap-tail write. `mkdir_one` step 6 has a "cap-tail owner stamp —
  DEFERRED to mkdir.M2-003" comment where the M2-003 placeholder
  invocation on `SLOT_USER` will slot in.
- No `CreatedDirRecord[]` emission. `mkdir_one` writes only stderr text
  via `sys_debug_puts` (`mkdir.M3-001`).
- No `libpdx-audit` journal. Every op currently emits nothing to the
  audit graph (`mkdir.M3-002`).
- No PdxFS v1 undo record. The RemoveRecord + `undo mkdir` machinery
  lands at `mkdir.M3-003`.
- No test matrix. Empty `tests/` tree until `mkdir.M4-001`.

## 9. Cross-repo dependencies

Per r49-r50-plan.md §5.9:

- **mkdir.M1** direct: paideia-os kernel through R48b (KIND_USER at 0x190,
  KIND_PDXFS_FILE at 0x195, KIND_PDXFS_TXN at 0x196, InitCap sidecar
  format at R20b.M4); libpdx-argv M1-002; libpdx-cap M1-001.
- **mkdir.M1** shape: shell.M4 (argv-in-sidecar wire); the argv linker-
  name convention we lean on at M1 is a transient shape.
- **mkdir.M2** substrate: create-dir op on KIND_PDXFS_FILE + open/commit
  ops on KIND_PDXFS_TXN (paideia-os follow-up filed as
  `mkdir.M2-substrate-001` at the parent repo when M1 closes).
- **mkdir.M3** libraries: libpdx-semantic-pipe M2, libpdx-audit M2.
- **mkdir.M5** distribution: pkg M4 (mirror + signature verification).

paideia-as ≥ v0.33 is required by the module encoder (for `mov_b` and
`@align`). Older paideia-as revisions predate the #1248 mitigation.
