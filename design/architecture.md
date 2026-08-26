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

> **Superseded by §4f (`mkdir.ENH-006`).** The signature above and
> Steps 4/7 below describe the M1 baseline, where `mkdir_one` itself
> opened and committed the TXN. As of `mkdir.ENH-006`, TXN open/commit
> live in `mkdir_run` instead (§4f) — `mkdir_one` no longer returns
> `MK_TXN_OPEN_FAIL` or `MK_TXN_COMMIT_FAIL`; those are `mkdir_run`
> return states now. Steps 1-3 and 6 below are still current.

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

## 4c. `mkdir_one` M2-002 additions — pre-existence probe

Step 6a inside the walk (was Step 6a "placeholder create" at M2-001)
now runs a pre-existence probe first. The M2-002 loop shape becomes:

- **6a. Pre-existence probe.** `sys_cap_invoke(SLOT_PARENT_FILE,
  PFF_OP_QUERY_INODE)`. Return convention (established at M2-002):
    - `rax == 1` → level exists on-disk. Set `comp_pre_existed[i] = 1`;
      skip create; advance `walk_i` without touching
      `newly_created_count`. This is the "no-op, not error" path the
      plan doc §5.9 M2-002 line requires.
    - `rax >= 0 && rax != 1` → level does not exist. Set
      `comp_pre_existed[i] = 0`; proceed to Step 6b.
    - `rax < 0` → hard failure. Return `MK_MKDIR_FAIL`.
- **6b. Placeholder create.** Unchanged from M2-001.
- **6c. `newly_created_count += 1`.** Now gated on Step 6a not hitting
  the "exists" branch — so the M3-003 undo record only queues levels
  this invocation actually created, preserving `undo mkdir -p a/b/c`
  correctness when `a` pre-existed.
- **6d. Cap-tail owner stamp.** Deferred to M2-003.

At M2-002 the probe is still a placeholder — the real per-component
existence check on `KIND_PDXFS_FILE` is `mkdir.M2-substrate-002` at
the paideia-os repo. The placeholder currently reports "not exists"
for every level, so the runtime behaviour of the M2-002 build is
indistinguishable from M2-001. The value the M2-002 change lands is
the branch structure: `comp_pre_existed[i]` bookkeeping, conditional
`newly_created_count` advancement, and the return-convention
contract (`rax == 1` = exists) that `mkdir.M2-substrate-002` fills
without a signature change to `mkdir_one`.

## 4d. `mkdir_one` M2-003 additions — cap-tail owner stamp

Step 6 grows a step 6d after the `newly_created_count` increment: a
per-create stamp of the invoker's `KIND_USER` cap (`SLOT_USER = 2`)
into the newly-created inode's owner tail. This is the mechanism by
which `ls --long <dir>` renders the correct owner (via
`libpdx-cap`'s cap-decode path) and by which the M3-001
`CreatedDirRecord`'s `owner:KIND_USER_ref` field gets a valid
reference.

- **6d. Cap-tail owner stamp.** `sys_cap_invoke(SLOT_USER,
  USER_OP_QUERY_FP0)`. Negative `rax` → `MK_CAP_TAIL_FAIL`. This is a
  hard error: a directory that was created but whose owner slot is
  unset would render as anonymous under `ls --long` — a D3/D4
  violation. TXN rollback of the already-created level is left to
  `mkdir.M2-substrate-001` (the placeholder cannot fail cleanly at
  M2 since `USER_OP_QUERY_FP0` is a QUERY op that reaches after the
  cap-invoke gate; the return-negative guard fires only against a
  future substrate wire that DOES fail conditionally).

The M2-003 placeholder invokes `USER_OP_QUERY_FP0` on `SLOT_USER` —
this exercises the KIND_USER slot dispatch (`kind_user.pdx` at
`src/kernel/core/cap/kind_user.pdx` checks `op_id <= USER_OP_MAX = 7`
per its handler at L1146) so a mis-seeded slot 2 surfaces at M2
rather than silently succeeding into an unowned inode. The real
"stamp-owner" op lives on `KIND_PDXFS_FILE` and takes `SLOT_USER` as
its cap argument — that dispatch is `mkdir.M2-substrate-003` at the
paideia-os repo. The M2-003 branch structure is what lands here; the
substrate wire fills the actual owner-write behind it.

## 4e. `mkdir_one` M3-001 additions — real substrate + CreatedDirRecord staging

M3-001 lands three coupled changes at `mkdir_one`:

1. **Substrate flip for Steps 6b + 7.** Step 6b's placeholder create
   was `sys_cap_invoke(SLOT_PARENT_FILE=1, PFF_OP_DEBUG_PRINT=6)` — a
   KIND_PDXFS_FILE-side op on the wrong cap kind. Flipped to
   `sys_cap_invoke(SLOT_TXN=0, PXT_OP_CREATE=9)`, the real R42-PREP-007
   (#1629) write-side op that stubs to `PXT_STUB_OK = 0xFFFFEBFB`
   (positive as i64 — passes the `jl` gate). Step 7's placeholder commit
   was `sys_cap_invoke(SLOT_TXN=0, PXT_OP_QUERY_STATE=4)`, a query op
   that never actually committed anything. Flipped to
   `sys_cap_invoke(SLOT_TXN=0, PXT_OP_COMMIT=7)`, the real state
   transition (OPEN → COMMITTED) landed by R42-PREP-007. Step 5 stays on
   `PXT_OP_QUERY_ID = 0` because mkdir's four-slot sidecar does not
   include a KIND_MEMORY mount cap to feed `sys_pdxfs_txn_open`
   (sysno 70) — the TXN cap is pre-opened by the shell and delivered via
   cap-transfer to `SLOT_TXN`.

2. **`current_txn_id` capture at Step 5.** `PXT_OP_QUERY_ID`'s return
   value is the pre-opened row's `txn_id`. M3-001 stashes it into
   `MkdirState::current_txn_id` for every `CreatedDirRecord`'s
   `parent_txn_id` field. All records in one invocation share the same
   `parent_txn_id` because the walk runs inside one TXN scope (per
   M2-001).

3. **`CreatedDirRecord[]` staging at Step 6e.** After Step 6d (cap-tail
   stamp succeeds), M3-001 writes one record into
   `MkdirState::created_dir_records` at index `newly_created_count - 1`.
   Fields (four u64 per record):

   | offset | field            | source                                              |
   |--------|------------------|-----------------------------------------------------|
   | +0     | `path_ptr`       | `rbx` (base of full path, preserved by push)        |
   | +8     | `path_len`       | `comp_start_offsets[walk_i] + comp_lengths[walk_i]` |
   | +16    | `parent_txn_id`  | `current_txn_id`                                    |
   | +24    | `owner_slot_ref` | `2` (== SLOT_USER, resolved by libpdx-cap M3-001)   |

   A three-level `mkdir -p a/b/c` with no pre-existing levels stages
   three records: `("a/b/c", 1, txn_id, 2)`, `("a/b/c", 3, txn_id, 2)`,
   `("a/b/c", 5, txn_id, 2)` — the same `path_ptr` with growing
   `path_len` names each level's prefix.

4. **`created_dir_records_count` freeze at Step 7.** Before the commit
   syscall, `mkdir_one` freezes
   `created_dir_records_count = newly_created_count`. This ordering
   ensures a commit failure (`PXT_BAD_TRANSITION` or a future substrate
   error) leaves the record array in the consistent "records staged,
   TXN never committed" state that `mkdir.M3-003`'s undo record replay
   reads to unwind exactly this invocation's additions.

The record schema handle is `CDR_SCHEMA_HANDLE = 0x0100000100010001`
(`CreatedDirRecord@0.1`), declared in `caps.decl` under
`declares_output_schemas`. `libpdx-semantic-pipe M2`'s `send_record`
consumer walks the staged records and marshals them through the
`KIND_IPC_ENDPOINT` at slot 3 — until that library lands, the records
live in `.bss` for the `M3-002` audit hook and the `M3-003` undo record
to read.

## 4f. `mkdir.ENH-006` — TXN lifecycle hoisted from `mkdir_one` to `mkdir_run`

The public synopsis (`README.md`, `doc/mkdir.pdxdoc`, this document's
§1) has always advertised `mkdir [-p] [-v] [--dry-run] <path> [<path>...]`,
and `mkdir_run` has always walked every positional. But through M3-003,
§4's Steps 4 (open) and 7 (commit) lived inside `mkdir_one`, which is
called ONCE PER POSITIONAL — so a TXN meant to span the whole
invocation was instead being opened and committed once per path:

- **Double commit.** Positional 2's Step 7 issued `PXT_OP_COMMIT`
  against the SAME pre-opened `SLOT_TXN` row that positional 1 had
  already transitioned to `COMMITTED`, hitting `PXT_BAD_TRANSITION`
  (negative) and failing `MK_TXN_COMMIT_FAIL`.
- **Record clobber.** Step 6 zeroed `newly_created_count` at the top of
  every `mkdir_one` call, and `CreatedDirRecord` / `RmdirUndoRecord`
  indices are `newly_created_count - 1` — so positional 2's records
  overwrote positional 1's at index 0, and positional 1's undo record
  was lost even before the double-commit failure surfaced.

`mkdir.ENH-006` fixes both by relocating the TXN open (§4 Step 4) and
the commit + count-freeze (§4 Step 7) out of `mkdir_one` and into
`mkdir_run`, which now:

1. Resets `newly_created_count` to 0 exactly ONCE, before the
   positional loop (not inside `mkdir_one` per call).
2. IF `flag_dry_run == 0`: opens ONE shared TXN
   (`sys_cap_invoke(SLOT_TXN, PXT_OP_QUERY_ID)`) and stashes
   `current_txn_id` once, before the loop. IF `flag_dry_run == 1`:
   skips this — every `mkdir_one` call short-circuits at its own Step 3
   before touching a cap, so there is nothing to open.
3. Calls `mkdir_one(path)` once per positional exactly as before; each
   call now performs ONLY Steps 1-3 + 6 (the create walk) and reads the
   already-populated `current_txn_id` — it neither opens nor commits.
   `newly_created_count` is a running total across every call in the
   invocation, so record indices append (`mkdir a b c` stages CDR/RUR
   entries at indices 0..2) instead of colliding at index 0.
4. On the first non-`MK_OK` return, records `exit_code` + `err_pos_index`
   and returns WITHOUT freezing the CDR/RUR counts or committing — the
   shared TXN is left uncommitted, so nothing this invocation staged is
   persisted (the same "staged, never committed" state §4e's Step 7
   already guaranteed on a single-positional failure, now guaranteed
   invocation-wide).
5. If every positional returns `MK_OK` and `flag_dry_run == 0`: freezes
   `created_dir_records_count` / `rmdir_undo_records_count` ==
   `newly_created_count` and issues exactly ONE
   `sys_cap_invoke(SLOT_TXN, PXT_OP_COMMIT)` for the whole invocation.

`mkdir_split_path`'s `mkdir.ENH-004` `..`-rejection guard (§4a) still
runs before any op that consumes the path being validated (Step 6's
probe/create/stamp) — the TXN-open call that now precedes it takes no
path argument and cannot navigate anywhere, so hoisting it ahead of
per-path validation does not weaken the containment guarantee.

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

## 8. What M2 (as a whole) explicitly does not do

M2-001/002/003 have all landed at HEAD. What remains queued:

- No real kernel-side ops. Every `sys_cap_invoke` in `mkdir_one`
  today is a QUERY / DEBUG_PRINT placeholder. The three substrate
  follow-ups at paideia-os fill them:
  - `mkdir.M2-substrate-001` — open-TXN / create-dir / commit-TXN
    ops on KIND_PDXFS_TXN + KIND_PDXFS_FILE.
  - `mkdir.M2-substrate-002` — per-component existence probe that
    returns 1 when the level exists, 0 otherwise.
  - `mkdir.M2-substrate-003` — stamp-owner op on KIND_PDXFS_FILE
    that writes the passed SLOT_USER cap reference into the just-
    created inode's owner tail.
- No `CreatedDirRecord[]` emission. `mkdir_one` writes only stderr
  text via `sys_debug_puts` (`mkdir.M3-001`).
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
