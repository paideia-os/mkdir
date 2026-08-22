# mkdir.M4-003 — implementation notes

**Issue:** #12 — TXN-abort mid-create: no dirs left.
**Upstream doc:** `design/tooling/r49-r50-plan.md` §5.9 (paideia-os).

## What landed

- `tests/test_m4_003_txn_abort.pdx` — `MkdirTestM4003` module with one
  driver `run_test_m4_003_abort`. Two-path structure:
    - Path A: happy-path atomicity baseline (`mkdir -p a/b/c` via full
      parse-run pipeline); asserts all four counts at 3 and
      `err_pos_index == 0`.
    - Path B: post-abort state shape freeze (seeds partial-completion
      + TXN-abort post-condition into .bss and reads back).
- `tests/expected-mkdir-m4-003-abort.txt` — `mkdir.M4-003 abort OK`
- `tests/README.md` + `STATUS.md` — M4-003 promoted to LANDED.

## The M2-001 atomicity invariant this test freezes

The plan doc §5.9 M4 line requires: "TXN-abort mid-create: no dirs
left". Mechanically, the property has TWO parts:

1. **TXN-scoped atomicity.** All levels in a single `-p` invocation
   run inside one `KIND_PDXFS_TXN` scope (M2-001). If any level's
   create returns negative, `mkdir_one_mkdir_fail` jumps to
   `mkdir_one_epilogue` and returns `MK_MKDIR_FAIL` WITHOUT executing
   Step 7's commit. The substrate's TXN abort (once
   `mkdir.M2-substrate-001` lands the real ABORT op) rolls back every
   create that had already succeeded in the same TXN scope.

2. **CDR/RUR count NOT frozen on failure.** The record-count freeze
   at Step 7 (`created_dir_records_count = rmdir_undo_records_count =
   newly_created_count`) runs BEFORE the commit syscall — but only on
   the success path. `mkdir_one_mkdir_fail` bypasses Step 7 entirely,
   so both counts stay at whatever `mkdir_state_reset` left them at
   (== 0). This means the M3-003 undo-log walker sees zero RURs for a
   failed TXN and skips replay entirely — nothing to undo because the
   TXN abort already rolled back the on-disk state.

The two properties together guarantee: after a mid-create failure,
neither the on-disk state (TXN aborted → nothing persisted) nor the
in-memory undo state (RUR count == 0 → no records to replay) points at
any surviving directory. That's the "no dirs left" invariant.

## Substrate gap at HEAD and the two-path workaround

The M3-001 real substrate `PXT_OP_CREATE` (op 9 on `SLOT_TXN`) STUBs
to `PXT_STUB_OK` (0xFFFFEBFB) — always succeeds. The real create body
that can return negative lives at `mkdir.M2-substrate-001` (paideia-os
follow-up). Until it lands, the walker cannot naturally take the
`mkdir_one_mkdir_fail` branch.

M4-003 handles this by splitting into two paths:

### Path A — happy-path atomicity baseline (runs at HEAD)

`mkdir -p a/b/c` through the normal bootstrap. All four count
invariants freeze at 3; `err_pos_index` stays at 0 because no
`mkdir_one` returned non-OK. Asserts:

- `exit_code == 0`
- `newly_created_count == 3`
- `created_dir_records_count == 3` (Step 7 freeze)
- `rmdir_undo_records_count == 3` (twin-freeze)
- `err_pos_index == 0`

Baseline for Path B's shape-freeze to be meaningful: if the walker
itself is broken, Path B's assertions are moot.

### Path B — post-abort state shape freeze (runs at HEAD)

Resets MkdirState, then seeds:

- `exit_code = MK_MKDIR_FAIL (5)`
- `newly_created_count = 1` (a created; b returned negative)

Does NOT touch `created_dir_records_count` or
`rmdir_undo_records_count` — the load-bearing invariant is that these
stay at 0 after `mkdir_state_reset` because Step 7 never executes on
the failure path. Then asserts:

- `exit_code == 5`
- `newly_created_count == 1`
- `created_dir_records_count == 0`
- `rmdir_undo_records_count == 0`

If Path B regresses, either the encoder moved the CDR/RUR slots, the
M3-001/M3-003 Step 7 twin-freeze became unconditional (bug — it MUST
only run on the success path), or `mkdir_state_reset` stopped zeroing
those counters (bug — they're both in the reset's justification-string
enumeration).

## Why `err_pos_index` is asserted 0 in both paths

`mkdir_run`'s record-fail branch stores `mkdir_run_i` (the failing
positional's index) into `err_pos_index` on the first non-OK
`mkdir_one` return. With ONE positional (`a/b/c`), `mkdir_run_i` is 0
when the fail record-store runs — so `err_pos_index == 0` in BOTH the
happy-path (never advanced) and the post-abort (advanced-to-0-then-
stopped) shapes. Testing this equality across both paths confirms the
first-fail short-circuit shape at HEAD.

## paideia-as conformance

- Module name PascalCase basename (`MkdirTestM4003`).
- No `test` mnemonic — every zero-check is `cmp reg, 0`.
- `r11` LEA scratch only.
- Every `cmp reg, imm` uses an immediate ≤ 5 (max: `exit_code == 5`
  assertion).
- No push/pop parity — straight-line driver.
- Labels prefixed `t4a_` (test-m4-003-abort). Avoids every paideia-as
  reserved keyword.
- Cross-module calls by unqualified linker name.

## What did NOT land at M4-003

- No substrate-driven mid-create failure. That requires
  `mkdir.M2-substrate-001` (real create op that can fail) plus a
  fault-injection hook to force the fail — filed at paideia-os as a
  future `mkdir.M4-substrate-003` issue when substrate lands.

## Cross-repo dependencies

- `mkdir.M4-003` direct: paideia-os kernel through R48b; libpdx-argv
  M1-002; `src/mkdir.pdx` at M3-003.
- `mkdir.M4-003` shape: `mkdir.M2-substrate-001` (real create op with
  failure return) + a fault-injection substrate for driving Path B
  end-to-end (both paideia-os follow-ups).

## Build note

No build here — main runs `bash tools/build.sh` and commits the
artefact once clean.
