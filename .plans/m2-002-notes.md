# mkdir.M2-002 — implementation notes

**Issue:** #5 — `-p` pre-existing dir handling (no-op, not error).
**Upstream doc:** `design/tooling/r49-r50-plan.md` §5.9 (paideia-os).

## What landed

- `src/mkdir_state.pdx`:
  - New `.bss` array `comp_pre_existed : [u64; 16]`. Consumed by
    index up to `comp_count`, so trailing entries need no zeroing at
    `mkdir_state_reset` — same discipline as `comp_start_offsets` /
    `comp_lengths` from M2-001.
  - `mkdir_state_reset` justification extended to mention M2-002.
- `src/mkdir.pdx`:
  - `mkdir_one` step 6 restructured: the "placeholder create" call
    (M2-001's step 6a) is now step 6b, preceded by a new M2-002
    step 6a — a pre-existence probe via
    `sys_cap_invoke(SLOT_PARENT_FILE, PFF_OP_QUERY_INODE)`.
  - Return-convention contract established at M2-002:
    - `rax == 1` → level exists on-disk. Set
      `comp_pre_existed[i] = 1`; skip create; advance `walk_i`
      without touching `newly_created_count`.
    - `rax >= 0 && rax != 1` → level does not exist. Set
      `comp_pre_existed[i] = 0`; proceed to create.
    - `rax < 0` → hard failure. Return `MK_MKDIR_FAIL`.
  - New labels: `mkdir_one_probe_exists`, `mkdir_one_do_create`,
    `mkdir_one_advance_walk`. All continue the `mkdir_one_` prefix
    convention; none clash with paideia-as reserved keywords
    (`advance`, `walk`, `probe`, `exists`, `create` are all safe).
  - `newly_created_count += 1` moved into the `mkdir_one_do_create`
    branch so pre-existing levels do NOT advance the counter — the
    M3-003 undo record reads this counter to unwind exactly this
    invocation's additions, preserving `undo mkdir -p a/b/c`
    correctness when `a` pre-existed.
  - File header + `mkdir_one` justification extended to reflect
    M2-002.
- `design/architecture.md`:
  - New §4c documents the M2-002 loop shape addition.
  - §8 updated: "What M2-002 explicitly does not do" replaces the
    M2-001 not-yet-done note about pre-existing dir handling.
- `STATUS.md` — M2-002 row marked LANDED.
- `.plans/m2-002-notes.md` (this file).

## Smoke witness at M2-002

The M2 bootstrap argv `-p -v --dry-run a/b/c` still short-circuits on
`--dry-run` before Step 5 (TXN open), so the smoke path is unchanged
from M2-001. The `--dry-run` branch never enters the walk loop and
never touches the probe. What M2-002 lands is the branch structure
that activates once `--dry-run` is dropped (a scenario the M4-002
test matrix will exercise once shell.M4 publishes real argv):

```
mkdir_run
  i=0: mkdir_one("a/b/c")
    Step 1  flag_p == 1                        → skip multi-level guard
    Step 2  mkdir_split_path("a/b/c")          → comp_count=3
    Step 4  flag_dry_run == 1                  → return MK_OK (0)
sys_exit(0)                                    → MK_OK smoke
```

The non-dry-run path (M4-002 test):

```
mkdir_one("a/b/c")   [with "a" pre-existing]
  Step 1  flag_p == 1                          → skip multi-level guard
  Step 2  mkdir_split_path                     → comp_count=3
  Step 4  flag_dry_run == 0                    → proceed
  Step 5  TXN open (placeholder)               → OK
  Step 6  walk:
    i=0: probe "a" → rax==1                    → comp_pre_existed[0]=1
                                                → skip create
    i=1: probe "b" → rax==0                    → comp_pre_existed[1]=0
                                                → create + newly_created=1
    i=2: probe "c" → rax==0                    → comp_pre_existed[2]=0
                                                → create + newly_created=2
  Step 7  TXN commit (placeholder)             → OK
  return MK_OK
```

`newly_created_count = 2` after this invocation. M3-003's undo record
will queue removal of "c" and "b" (not "a"). This is the
`design/tooling/r49-r50-plan.md` §5.9 M2-002 line invariant literally.

At M2-002 with the substrate placeholder, the probe never returns 1
so every level takes the create branch. M2-substrate-002 flips the
probe to return 1 for pre-existing levels; no signature change to
`mkdir_one` is needed at that landing.

## Design decisions

- **Probe placed BEFORE create, not after.** Alternative was to
  attempt the create and treat `EEXIST` as no-op. The probe-first
  shape mirrors how the kernel-side create will report EEXIST at the
  substrate landing (a per-op dispatch check would still preserve
  the placeholder shape). Probe-first also lets the M2-002 branch
  land cleanly without waiting on a "create returns positive vs
  EEXIST" convention that would itself be a substrate concern.
- **`comp_pre_existed[i] = 1` stored explicitly.** Alternative was to
  leave the entry uninitialised for pre-existing levels and rely on
  the caller to check `walk_i < newly_created_count`. Explicit
  storage lets M3-003's undo builder iterate `0..comp_count` and
  read the bit per-level rather than reconstructing which levels
  were skipped from a running counter. Matches the D3 invariant
  "every operation is discoverable" — every level in the walk has
  an explicit pre_existed bit.
- **`newly_created_count` advancement gated at Step 6c, not 6a.**
  Alternative was to advance in the "exists" branch too and let
  M3-003 subtract. Gating at 6c is simpler: the counter directly
  answers "how many levels does undo need to remove", no
  post-processing.
- **Probe placeholder shares SLOT_PARENT_FILE with create.** The
  probe dispatches through the same cap as the create — same slot,
  different op ordinal. This means a mis-seeded slot 1 surfaces at
  the first probe (immediately when Step 6 begins) rather than at
  the first create. No new cap slot needed.
- **Rax == 1 sentinel for "exists".** Alternative was to use `rax > 0`
  as "exists". Sticking with `rax == 1` leaves higher return values
  reserved for future substrate use (e.g., 2 = "exists with
  different owner", would eventually route into a permissions
  branch). The `rax < 0` branch still catches every hard error.
- **Placeholder probe uses `PFF_OP_QUERY_INODE = 0`.** Alternative
  was `PFF_OP_QUERY_MODE = 2` (which more naturally represents "is
  this a directory"). Choosing `QUERY_INODE` matches the M2-001
  create placeholder's own op ordinal choice (using ordinals ≥ MAX
  would trigger the kernel's `PFF_OP_MAX` check). `QUERY_INODE`
  exercises the KIND_PDXFS_FILE dispatch through the same slot the
  create uses — enough to sanity-check the cap seed at M2-002.

## paideia-as conformance

- No `test` mnemonic anywhere in the new code.
- Every `cmp reg, imm` uses an immediate ≤ 0x7FFFFFFF (max seen at
  M2-002: `1` for the pre-existence sentinel, `0` for the
  fail-check).
- No byte loads added at M2-002 (the probe returns a full u64 rax
  from `sys_cap_invoke`; no `mov_b` needed). Existing `mov_b` sites
  from M1/M2-001 unchanged.
- `r11` used only as scratch (LEA temps for `.bss` slot addresses).
  `rcx` briefly holds `walk_i` across a single `mov` chain but is
  never live across a `syscall` or `call`.
- `mkdir_one` push/pop parity unchanged from M2-001: one `push rbx`
  in the prologue matched by one `pop rbx` at the single epilogue
  label. All new labels route through `mkdir_one_epilogue` or fall
  through the walk-advance path; no early `ret` bypasses the pop.
- Labels avoid every paideia-as reserved keyword. New labels:
  `mkdir_one_probe_exists`, `mkdir_one_do_create`,
  `mkdir_one_advance_walk` — none clash with `loop`, `if`, `let`,
  `fn`, `pub`, `mut`, `struct`, `structure`, `unsafe`, `block`.
- SysV alignment: no new `call` sites in Step 6a; the added
  `syscall` inherits the same alignment discipline as the existing
  create-syscall from M2-001.

## Cross-module linkage

New unqualified linker-name reference introduced by M2-002:
- `comp_pre_existed` → local `MkdirState` (`.bss` reads/writes).

No new cross-repo dependencies. The probe reuses `SLOT_PARENT_FILE`
(caps.decl slot 1, unchanged) and `PFF_OP_QUERY_INODE` (a HEAD-live
op ordinal on `KIND_PDXFS_FILE`).

## What did not land (queued for M2-003 and beyond)

- Cap-tail owner write — `mkdir.M2-003` (#6). Adds a placeholder
  `sys_cap_invoke(SLOT_USER, USER_OP_QUERY_FP0)` after each
  successful create; `MK_CAP_TAIL_FAIL` return code added.
- Real per-component existence probe — `mkdir.M2-substrate-002` at
  paideia-os. Fills the `rax == 1` return convention for
  pre-existing per-component levels.
- Kernel-side create-dir / open-TXN / commit-TXN ops —
  `mkdir.M2-substrate-001` at paideia-os.
- `CreatedDirRecord[]` schema bind — `mkdir.M3-001`.
- `CreateDirRecord` via libpdx-audit — `mkdir.M3-002`.
- PdxFS v1 undo record — `mkdir.M3-003` (reads
  `newly_created_count` + `comp_pre_existed[]` to build the RemoveRecord).
- Coreutil test matrix — `mkdir.M4-001` through `mkdir.M4-004`,
  including M4-002 "mixed pre-existing + new under -p".

## Build note

Same as M2-001: builds run main-only per the no-background-builds
memory note; this issue lands the source and defers the build check
to main's synchronous `bash tools/build.sh` step.
