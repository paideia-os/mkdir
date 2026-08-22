# mkdir.M3-001 — implementation notes

**Issue:** #7 — `CreatedDirRecord[]` schema bind (path, parent_txn_id, owner).
**Upstream doc:** `design/tooling/r49-r50-plan.md` §5.9 (paideia-os).

## What landed

- `src/mkdir_state.pdx`:
  - New op-code constants for the substrate flip:
    `PXT_OP_COMMIT = 7`, `PXT_OP_CREATE = 9` (R42-PREP-007 #1629 at
    `src/kernel/core/cap/kind_pdxfs_txn.pdx` L115-119).
  - `CDR_FIELDS_PER_RECORD = 4`, `CDR_MAX_RECORDS = 16`.
  - `CDR_SCHEMA_HANDLE = 0x0100000100010001` — the on-wire handle
    for `CreatedDirRecord@0.1` (declared in `caps.decl` under
    `declares_output_schemas`).
  - New `.bss` slots:
    - `current_txn_id : u64` — populated at Step 5 from
      `PXT_OP_QUERY_ID`; consumed by every record's `parent_txn_id`.
    - `created_dir_records : [u64; 64]` — row-major array of 16
      records × 4 fields (`path_ptr`, `path_len`, `parent_txn_id`,
      `owner_slot_ref`).
    - `created_dir_records_count : u64` — set at commit time from
      `newly_created_count`; consumers walk `[0..count)`.
  - `mkdir_state_reset` extended to zero the two new counters
    (`current_txn_id`, `created_dir_records_count`); the record array
    is consumed by index up to the counter, so trailing entries are
    unreachable and stay uninitialized.

- `src/mkdir.pdx`:
  - **Substrate flip #1 (Step 6b create).** Was
    `sys_cap_invoke(SLOT_PARENT_FILE=1, PFF_OP_DEBUG_PRINT=6)` — a
    KIND_PDXFS_FILE-side op on the wrong cap kind. Now
    `sys_cap_invoke(SLOT_TXN=0, PXT_OP_CREATE=9)` — real
    R42-PREP-007 op that STUBs to `PXT_STUB_OK = 0xFFFFEBFB`. The
    stub value is positive as signed i64 (top nibble is 0x0), so
    the existing `cmp rax, 0; jl mkdir_one_mkdir_fail` gate still
    passes cleanly on stub success.
  - **Substrate flip #2 (Step 7 commit).** Was
    `sys_cap_invoke(SLOT_TXN=0, PXT_OP_QUERY_STATE=4)` — a query
    that never actually committed. Now
    `sys_cap_invoke(SLOT_TXN=0, PXT_OP_COMMIT=7)` — real
    OPEN → COMMITTED transition. First-commit succeeds (PXT_OK = 0);
    a hypothetical duplicate commit would return
    `PXT_BAD_TRANSITION = 0xFFFFEC15` (negative-shaped, top nibble
    0xF) and route to `mkdir_one_commit_fail`. mkdir is one-shot per
    process, so the duplicate-commit edge is unreachable at v1.0.
  - **Step 5 (TXN "open") NOT flipped.** Stays on
    `sys_cap_invoke(SLOT_TXN=0, PXT_OP_QUERY_ID=0)`. mkdir's four-
    slot sidecar does not include a KIND_MEMORY mount cap to feed
    `sys_pdxfs_txn_open` (sysno 70 at
    `src/kernel/core/syscall/handlers/sys_pdxfs_txn_open.pdx`); the
    TXN cap is instead pre-opened by the shell and delivered via
    cap-transfer to SLOT_TXN. `PXT_OP_QUERY_ID` reads the pre-opened
    row's `txn_id`, which M3-001 captures into `current_txn_id` for
    the record's `parent_txn_id` field.
  - **Step 6e (new) — record staging.** After every successful
    create + stamp, `mkdir_one` writes one `CreatedDirRecord` into
    `created_dir_records` at index `newly_created_count - 1`.
    Fields:
    - `path_ptr` = `rbx` (the base of the full path string,
      preserved across every syscall by the mkdir_one prologue push)
    - `path_len` = `comp_start_offsets[walk_i] +
      comp_lengths[walk_i]` — the byte count from `path_ptr` that
      names THIS level's full prefix.
    - `parent_txn_id` = `current_txn_id` (all records in one
      invocation share this).
    - `owner_slot_ref` = `2` (== SLOT_USER); libpdx-cap M3-001's
      decode helper resolves the slot to a `user_fp_lo` at render
      time.
  - **Step 7 — record count freeze.** Before the commit syscall,
    `mkdir_one` sets `created_dir_records_count = newly_created_count`.
    This ordering leaves the record array consistent even if commit
    fails: the records staged before commit persist so M3-003's undo
    record replays exactly the levels that were created before the
    abort.
  - `mkdir_one` justification string extended with the M3-001 tag
    and a description of the three coupled changes.

- `design/architecture.md`:
  - New §4e "M3-001 additions" documents the substrate flip,
    `current_txn_id` capture, `CreatedDirRecord[]` staging (with the
    field-source table and the a/b/c walk example), and the
    `created_dir_records_count` freeze ordering.

- `STATUS.md`: M3-001 row LANDED; current milestone bumped to M3
  in-progress with M3-002 / M3-003 QUEUED.

- `.plans/m3-001-notes.md` (this file).

## Smoke witness at M3-001

The bootstrap argv `-p -v --dry-run a/b/c` still short-circuits on
`--dry-run` before Step 5, so M3-001's Steps 5-7 (real TXN ops +
record staging) are inert during the smoke — the exit-0 witness is
unchanged from M2-003. What M3-001 lands is what fires once
`--dry-run` is dropped:

```
mkdir_one("a/b/c") [no --dry-run, no pre-existing]
  Step 1  flag_p == 1                          → skip guard
  Step 2  mkdir_split_path                     → comp_count=3
  Step 4  flag_dry_run == 0                    → proceed
  Step 5  TXN open: PXT_OP_QUERY_ID            → rax=txn_id
          store current_txn_id = txn_id
  Step 6  walk:
    i=0: probe → 0 → comp_pre_existed[0]=0
         create: PXT_OP_CREATE on SLOT_TXN     → PXT_STUB_OK (positive)
         newly_created_count=1
         cap-tail stamp SLOT_USER              → OK
         record[0] = ("a/b/c", 1, txn_id, 2)
    i=1: probe → 0 → comp_pre_existed[1]=0
         create: PXT_OP_CREATE on SLOT_TXN     → PXT_STUB_OK
         newly_created_count=2
         cap-tail stamp                        → OK
         record[1] = ("a/b/c", 3, txn_id, 2)
    i=2: probe → 0 → comp_pre_existed[2]=0
         create: PXT_OP_CREATE on SLOT_TXN     → PXT_STUB_OK
         newly_created_count=3
         cap-tail stamp                        → OK
         record[2] = ("a/b/c", 5, txn_id, 2)
  Step 7  created_dir_records_count = 3
          TXN commit: PXT_OP_COMMIT on SLOT_TXN → PXT_OK (0)
  return MK_OK
```

Each record's `path_len` is the byte count from `path_ptr` that
names the level's full prefix — path 1 = "a", path 3 = "a/b",
path 5 = "a/b/c". The three records collectively describe the
tree M3-003's undo record must reverse under `undo mkdir -p a/b/c`.

At M4-001 the coreutil test matrix's record-staging check will
assert exactly this shape: after a three-level `-p` create with no
pre-existing levels, `created_dir_records_count == 3` and the
three records name the growing-prefix path family.

## Design decisions

- **Real PXT_OP_CREATE + PXT_OP_COMMIT even though the walker
  body isn't landed.** The user directive at M3 open was to flip
  the three stubbed substrate paths where the substrate op exists
  today. R42-PREP-007 landed CREATE (9) as a stub (PXT_STUB_OK)
  and COMMIT (7) as a real state transition; both are safe to
  wire because their return values pass the existing `jl` gate
  and their side effects at v1.0 are exactly what mkdir expects
  (CREATE stub bumps `PXT_ST_CREATES` so `pdxfs_txn_stat(6)`
  counts real mkdir invocations; COMMIT transitions the pre-
  opened TXN row's state so a duplicate commit would fail cleanly).
  Waiting for `mkdir.M2-substrate-001` to flip would leave M3-001
  reading `current_txn_id` from a query that has no relationship
  to any actual write — the real TXN's `txn_id` is exactly what
  the record needs.

- **Step 5 stays on `PXT_OP_QUERY_ID`, not `sys_pdxfs_txn_open`.**
  The syscall path (sysno 70) requires a KIND_MEMORY parent cap
  with `RIGHT_MINT` (per `pdxfs_txn_check_parent_memory` at
  `kind_pdxfs_txn.pdx` L275). mkdir's four-slot sidecar carries
  `KIND_PDXFS_TXN` at slot 0, `KIND_PDXFS_FILE` at slot 1,
  `KIND_USER` at slot 2, `KIND_IPC_ENDPOINT` at slot 3 — no
  KIND_MEMORY. Adding a fifth cap slot for a mount authority
  would be a caps.decl change that widens mkdir's authority beyond
  what §5.9 authorises ("no elevate dependency at v1.0" is the
  strong claim; carrying a mount cap for every invocation would
  mean every mkdir could open TXNs against any file in the
  invoker's subtree, which mkdir does not otherwise need). The
  shell-owns-TXN pattern keeps mkdir's four-cap invocation
  contract; `sys_pdxfs_txn_open` is the pattern for tools like
  `pkg` and `cp` whose §5.1 / §5.6 caps.decl DO carry a mount
  cap.

- **Record fields are the schema-plan minimum.** §5.9 M3-001
  says "fields `{path, parent_txn_id, owner:KIND_USER_ref}`" —
  three fields. Staging as four (`path_ptr` + `path_len` split;
  `owner_slot_ref` for the KIND_USER_ref) gives libpdx-semantic-
  pipe M2's `send_record` a fixed-width record it can copy in
  one memcpy per record without a variable-length path indirection
  at marshal time. The library's `KIND_USER_ref` decoder
  (libpdx-cap M3-001) resolves the `owner_slot_ref` to a real
  user_fp_lo when the record is rendered, so the on-wire
  representation still carries a user-fingerprint reference —
  matches the schema-plan intent.

- **`path_ptr` = base of full path; `path_len` = growing prefix.**
  Alternative was per-level path strings copied out. Base + length
  needs zero allocation: every level's path is a prefix of the
  argv-provided string, so we point at the same bytes with a
  different length. The M4-001 test matrix will assert the byte
  sequence `path_ptr[0..path_len]` names each level correctly;
  M3-002's audit hook and M3-003's undo record read the same
  base + length pair without duplicating the string.

- **`created_dir_records_count` set BEFORE commit, not after.**
  The R42-PREP-007 commit substrate is one-shot (OPEN → COMMITTED
  is irreversible); on failure the records staged before the
  attempt still exist on-disk (once the mkdir.M2-substrate-001
  walker body lands and the STUB flips to real). Setting the
  count before commit means M3-003's undo record replay reads a
  count that matches the on-disk state, so `undo mkdir` after a
  commit failure removes exactly what got created — not one
  more, not one less. If the count were set after commit,
  a commit-failure invocation would leave `created_dir_records_count
  = 0` while `newly_created_count = comp_count`, and the undo
  record would replay zero levels, leaving partial-create debris
  on disk.

- **No push/pop parity change to `mkdir_one`.** The record
  staging block is straight-line — six memory stores through r11
  scratch — with no `call` and no callee-save touched. The
  prologue's single `push rbx` still balances the epilogue's
  single `pop rbx`; every non-OK return still routes through
  `mkdir_one_epilogue`.

## paideia-as conformance

- No `test` mnemonic anywhere in the new code.
- Every `cmp reg, imm` uses an immediate ≤ 0x7FFFFFFF (max seen at
  M3-001: `7` for PXT_OP_COMMIT syscall op-arg, `9` for
  PXT_OP_CREATE, `2` for SLOT_USER stored into `owner_slot_ref`).
- No byte loads added at M3-001 (record staging uses u64 stores
  through `mov [addr], reg`).
- `r11` used only as scratch (LEA temp for every `.bss` slot lookup
  in the record-staging block).
- `mkdir_one` push/pop parity unchanged: still one `push rbx` /
  one `pop rbx`.
- Labels avoid every paideia-as reserved keyword. No new labels
  added at M3-001 — the record-staging block is straight-line
  inside `mkdir_one_do_create` between the cap-tail stamp and
  `mkdir_one_advance_walk`, and the commit-count freeze is
  straight-line at the head of `mkdir_one_do_commit`.
- SysV alignment: no new `call` sites in the record-staging block
  (all memory stores are syscall-free), so alignment discipline
  from the caller (prologue's `push rbx` aligns rsp to 16) is
  preserved through the epilogue's `pop rbx`.

## Cross-module linkage

New unqualified linker-name references introduced by M3-001:
- `current_txn_id` → local (`.bss` read at Step 6e, write at Step 5).
- `created_dir_records` → local (`.bss` writes at Step 6e).
- `created_dir_records_count` → local (`.bss` write at Step 7 head).

No new cross-repo dependencies. The substrate flip references
`PXT_OP_CREATE = 9` and `PXT_OP_COMMIT = 7` — both are HEAD-live
op ordinals on `KIND_PDXFS_TXN` (R42-PREP-007 #1629).

## What did not land (queued for M3-002 + M3-003)

- `emit_audit_tool_invoke` / `emit_audit_create_dir_record` / audit
  slot in `_init_caps` — `mkdir.M3-002`. Will add a 5th cap slot
  (KIND_IPC_ENDPOINT bound to `svc.audit-journal`) and emit
  `UEJ_KIND_TOOL_INVOKE (130)` at _start entry, one
  `UEJ_KIND_TOOL_CREATE_DIR` (or the shape libpdx-audit picks) per
  successful create, and `UEJ_KIND_TOOL_ERROR (131)` on non-zero
  exit. The kernel-side seam
  (`src/kernel/core/ipc/audit_journal_broker.pdx`) already exposes
  `audit_journal_broker_dispatch(event_kind, payload_ptr)` returning
  `AJB_DISPATCH_STUB (0xFFFFEBEB)` for any recognised kind.

- `UndoRecord[]` + `emit_undo_record` — `mkdir.M3-003`. Will
  populate an undo record per successful create (fields
  `{path_ptr, path_len, parent_txn_id}`) so `undo mkdir` replays
  as `PXT_OP_UNLINK (11)` on each staged path.

- `libpdx-semantic-pipe` `send_record` consumer — the actual
  emission of `CreatedDirRecord[]` over the KIND_IPC_ENDPOINT at
  slot 3 waits on `libpdx-semantic-pipe.M2`. M3-001 lands the
  record staging shape that the consumer will read.

- Coreutil test matrix — `mkdir.M4-001` through `mkdir.M4-004`,
  including a M4-001 record-staging check that asserts the three-
  record shape for `-p a/b/c` and the growing-prefix path family.

## Build note

Same as prior milestones: builds run main-only per the
no-background-builds memory note; this issue lands the source and
defers the build check to main's synchronous `bash tools/build.sh`
step.
