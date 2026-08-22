# mkdir.M2-003 — implementation notes

**Issue:** #6 — cap-tail write on every created directory
(`KIND_USER_ref` in inode).
**Upstream doc:** `design/tooling/r49-r50-plan.md` §5.9 (paideia-os).

## What landed

- `src/mkdir_state.pdx`:
  - New return code `MK_CAP_TAIL_FAIL = 9`.
- `src/mkdir.pdx`:
  - New stderr diagnostic `err_cap_tail_msg` (29 B, padded to 32).
  - `mkdir_one` step 6d activated: placeholder cap-tail owner stamp
    `sys_cap_invoke(SLOT_USER, USER_OP_QUERY_FP0)` after every
    successful create. Negative `rax` routes to a new
    `mkdir_one_cap_tail_fail` label that emits the diagnostic and
    returns `MK_CAP_TAIL_FAIL` through the shared epilogue.
  - File header + `mkdir_one` justification extended to reflect
    M2-003. Note added that `USER_OP_QUERY_FP0` is a QUERY op
    (cannot fail cleanly at M2) — the guard exists for the future
    substrate wire (`mkdir.M2-substrate-003`) which will use a
    stamp op on `KIND_PDXFS_FILE` taking `SLOT_USER` as its cap
    argument.
- `design/architecture.md`:
  - New §4d documents the M2-003 stamp step.
  - §8 rewritten as "What M2 (as a whole) explicitly does not do"
    now that all three M2 issues have landed. Only the three
    kernel-side substrate follow-ups (`mkdir.M2-substrate-001/002/003`)
    and the M3 wave items remain.
- `STATUS.md` — M2 marked CLOSED; M2-003 row LANDED. Current
  milestone bumped to M3 (semantic-pipe / audit integration).
- `.plans/m2-003-notes.md` (this file).

## Smoke witness at M2-003

The bootstrap argv `-p -v --dry-run a/b/c` still short-circuits on
`--dry-run` before Step 5, so M2-003's new step 6d is inert during
the smoke. What M2-003 lands is the branch that fires once
`--dry-run` is dropped:

```
mkdir_one("a/b/c") [no --dry-run, no pre-existing]
  Step 1  flag_p == 1                          → skip guard
  Step 2  mkdir_split_path                     → comp_count=3
  Step 4  flag_dry_run == 0                    → proceed
  Step 5  TXN open (placeholder)               → OK
  Step 6  walk:
    i=0: probe → 0 → comp_pre_existed[0]=0
         create → OK; newly_created_count=1
         cap-tail stamp SLOT_USER → OK
    i=1: probe → 0 → comp_pre_existed[1]=0
         create → OK; newly_created_count=2
         cap-tail stamp SLOT_USER → OK
    i=2: probe → 0 → comp_pre_existed[2]=0
         create → OK; newly_created_count=3
         cap-tail stamp SLOT_USER → OK
  Step 7  TXN commit (placeholder)             → OK
  return MK_OK
```

Every level receives a cap-tail stamp. At M4-004 ("cap-tail
correctness (owner matches invoker in every created inode)") the
test matrix compares each created inode's owner tail against the
invoker's KIND_USER cap; M2-003 lands the stamp mechanism so M4-004
has something to check.

## Design decisions

- **Stamp AFTER `newly_created_count` increment, not before.** A
  pre-existing level (M2-002 branch) is not stamped because we
  never reach the `mkdir_one_do_create` branch. Placing the stamp
  after the increment keeps the two coupled — every advance of
  `newly_created_count` has a matching stamp. M3-003's undo record
  iterates `newly_created_count` levels and reverses each by
  removing the directory; the stamp does not need explicit undo
  because removing the directory implicitly unlinks its owner tail.
- **Placeholder invokes `USER_OP_QUERY_FP0` on `SLOT_USER`, not a
  stamp op on `SLOT_PARENT_FILE`.** The real stamp op takes two
  caps (SLOT_PARENT_FILE as the target inode + SLOT_USER as the
  owner reference). At M2 we have neither a stamp-op ordinal on
  `KIND_PDXFS_FILE` nor a way to pass a second cap argument. The
  placeholder exercises SLOT_USER on its own — enough to prove
  slot 2 is seeded with a KIND_USER cap of the correct kind (the
  kernel's `kind_user.pdx` dispatch checks `op_id <= USER_OP_MAX`
  and refuses on kind mismatch). The substrate wire
  (`mkdir.M2-substrate-003`) replaces this with the two-cap stamp
  op on SLOT_PARENT_FILE without a signature change.
- **Stamp failure is a hard error.** Alternative was to treat
  `MK_CAP_TAIL_FAIL` as a warning and continue. A directory without
  a recorded owner would surface as anonymous under `ls --long` —
  a D3 (audit-first) + D4 (signed cap-tail) violation. Making it
  hard means M4-004's test matrix asserts stamp success on every
  created level; a substrate regression that broke the stamp would
  fail M4-004 rather than passing silently with a broken audit
  trail.
- **New label `mkdir_one_cap_tail_fail` placed in the same
  error-block cluster as the other `*_fail` labels.** Alternative
  was to emit + return inline at the stamp call site. The cluster
  form matches the M1/M2-001 shape (`mkdir_one_txn_open_fail`,
  `mkdir_one_mkdir_fail`, `mkdir_one_commit_fail`,
  `mkdir_one_abs_path_fail`, `mkdir_one_too_deep_fail`) — the
  reader can grep for `_fail` and see every error return in one
  place. The `mkdir_one_epilogue` label is the single `ret` site.
- **`err_cap_tail_len = 29` (padded to 32).** Same padding
  discipline as `err_multi_msg` / `err_abs_path_msg`: the
  `[u8; N]` type is rounded up so the storage stays 8-byte
  aligned; the paired `*_len` constant is the exact byte count
  passed to `sys_debug_puts`.

## paideia-as conformance

- No `test` mnemonic anywhere in the new code.
- Every `cmp reg, imm` uses an immediate ≤ 0x7FFFFFFF (max seen at
  M2-003: `0` for the stamp fail-check, `29` for the diagnostic
  length).
- No byte loads added at M2-003 (the stamp uses full u64 rax from
  `sys_cap_invoke`).
- `r11` used only as scratch (LEA temp for `err_cap_tail_msg` and
  `mkdir_one_cap_tail_fail` epilogue routing).
- `mkdir_one` push/pop parity unchanged: still one `push rbx` /
  one `pop rbx`. The new `mkdir_one_cap_tail_fail` label routes
  through `mkdir_one_epilogue` — no early `ret` bypasses the pop.
- Labels avoid every paideia-as reserved keyword. New label:
  `mkdir_one_cap_tail_fail` — `cap`, `tail`, `fail` are all safe
  (not in the reserved list `loop`, `if`, `let`, `fn`, `pub`,
  `mut`, `struct`, `structure`, `unsafe`, `block`).
- SysV alignment: the new `syscall` inherits the same alignment
  discipline as the existing probe- and create-syscalls in Step 6.
  Prologue's `push rbx` still aligns rsp to 16 for the `call
  emit_stderr` inside `mkdir_one_cap_tail_fail`.

## Cross-module linkage

New unqualified linker-name references introduced by M2-003:
- `err_cap_tail_msg` → local (`.rodata` read from
  `mkdir_one_cap_tail_fail`).

No new cross-repo dependencies. The stamp reuses `SLOT_USER`
(caps.decl slot 2, unchanged) and `USER_OP_QUERY_FP0` (a HEAD-live
op ordinal on `KIND_USER`).

## What did not land (queued for M3 wave)

- `CreatedDirRecord[]` schema bind + emit on `-v` — `mkdir.M3-001`.
  The `owner:KIND_USER_ref` field this schema carries is what
  M2-003's cap-tail stamp feeds; the record's marshal will read the
  same `SLOT_USER` cap that M2-003 stamps into the inode.
- `CreateDirRecord` via libpdx-audit — `mkdir.M3-002`.
- PdxFS v1 undo record — `mkdir.M3-003` (reads
  `newly_created_count` + `comp_pre_existed[]` to build the
  RemoveRecord; the stamp does not need explicit undo — removing
  the level unlinks its owner tail).
- Kernel-side real ops — `mkdir.M2-substrate-001/002/003` at
  paideia-os.
- Coreutil test matrix — `mkdir.M4-001` through `mkdir.M4-004`,
  including M4-004 "cap-tail correctness (owner matches invoker in
  every created inode)" which asserts the stamp shape M2-003 lands.

## Build note

Same as M2-002: builds run main-only per the no-background-builds
memory note; this issue lands the source and defers the build check
to main's synchronous `bash tools/build.sh` step.
