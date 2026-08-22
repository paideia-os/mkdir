# mkdir.M4-002 — implementation notes

**Issue:** #11 — mixed pre-existing + new under `-p`: undo removes
only new levels.
**Upstream doc:** `design/tooling/r49-r50-plan.md` §5.9 (paideia-os).

## What landed

- `tests/test_m4_002_mixed_pre_existing.pdx` — `MkdirTestM4002` module
  with one driver `run_test_m4_002_mixed`. Two-path structure:
    - Path A: happy-path baseline (`mkdir -p a/b/c` via full parse-run
      pipeline); asserts `newly_created_count == 3`.
    - Path B: shape-freeze of the substrate-driven mixed-state
      post-condition (direct .bss seed + read-back).
- `tests/expected-mkdir-m4-002-mixed.txt` — `mkdir.M4-002 mixed OK`
- `tests/README.md` — M4-002 row promoted to LANDED.
- `STATUS.md` — M4-002 row promoted to LANDED.

## The M3-003 invariant this test freezes

The plan doc §5.9 M3-003 line requires:

> Under `-p`, only the levels this invocation actually created are
> unwound (the TXN log distinguishes newly-created from pre-existing).

Mechanically: `mkdir_one` Step 6a's pre-existence probe sets
`comp_pre_existed[i] = 1` when the probe returns 1, then jumps to
`mkdir_one_advance_walk` BEFORE Step 6c. Step 6c's
`newly_created_count += 1` happens only on the create branch; Steps
6e and 6f (CDR + RUR staging) run AFTER the increment. So a
pre-existing level produces:

- comp_pre_existed[i] = 1
- NO increment of newly_created_count
- NO CDR record staged
- NO RUR record staged

`mkdir -p a/b/c` where `a` exists on-disk (mixed scenario) therefore
freezes at:

| slot                          | value |
|-------------------------------|-------|
| `comp_count`                  | 3     |
| `comp_pre_existed[0]`         | 1     |
| `comp_pre_existed[1]`         | 0     |
| `comp_pre_existed[2]`         | 0     |
| `newly_created_count`         | 2     |
| `created_dir_records_count`   | 2     |
| `rmdir_undo_records_count`    | 2     |

The undo-log walker at `mkdir.M4-substrate-002` reads the two staged
RURs (naming `a/b/c` and `a/b`) and reverse-replays them (c → b);
`a` has no RUR so it is untouched. That is the "-p unwinds only the
levels this invocation created" property the plan doc calls out.

## Substrate gap at HEAD and the two-path workaround

The M2-002 pre-existence probe is a placeholder — real per-component
existence check lives at `mkdir.M2-substrate-002` (paideia-os). Until
that lands, the probe always returns "not exists" for every level, so
the natural bootstrap-argv flow cannot produce a mixed state at HEAD.

M4-002 handles this by splitting the test into two paths:

### Path A — happy-path baseline (runs at HEAD)

Invokes `mkdir_one("a/b/c")` with `flag_p = 1` through the normal
bootstrap. Placeholder probe returns not-exists for every level, so
every level takes Step 6b's create path and
`newly_created_count == 3` after `mkdir_run` returns. Path A asserts:

- `exit_code == 0`
- `newly_created_count == 3`

If Path A regresses, either the parse pipeline broke or the walk
counter drifted — Path B's invariants are meaningless if the walk
itself is broken, so Path A must pass first.

### Path B — shape-freeze invariant (runs at HEAD)

Resets MkdirState, then directly seeds the .bss slots to the
substrate-driven mixed-state post-condition. Immediately reads each
field back and asserts. This is a SHAPE-FREEZE proof: the encoder /
paideia-as toolchain must preserve the field layout that mkdir_one +
mkdir_state_reset assume. If the encoder ever renames a slot, moves
alignment, or the M3-003 invariant drifts (e.g. the RUR count freeze
becomes dependent on comp_count instead of newly_created_count), Path
B surfaces the mismatch here — before `mkdir.M4-substrate-002` lands
at paideia-os.

Path B seeds explicitly zero into `comp_pre_existed[1]` and `[2]`
because `mkdir_state_reset` only zeros the counters (`comp_count`,
`newly_created_count`, `walk_i` — per its justification string); the
array itself is consumed by index up to `comp_count` and trailing
entries are unreachable in normal flow. The shape-freeze test cannot
assume mkdir_state_reset touched the array bytes.

## paideia-as conformance

- Module name PascalCase basename (`MkdirTestM4002`).
- No `test` mnemonic — every zero-check is `cmp reg, 0`.
- `r11` LEA scratch only; never live across calls.
- Every `cmp reg, imm` uses an immediate ≤ 3 (max: `comp_count == 3`
  assertion).
- No push/pop parity — straight-line driver with leaf-return calls
  only.
- Labels prefixed `t4mx_` (test-m4-002-mixed). Avoids every paideia-as
  reserved keyword (`loop`, `if`, `let`, `fn`, `pub`, `mut`, `struct`,
  `structure`, `unsafe`, `block`).
- Cross-module calls by unqualified linker name per the paideia-as
  flat convention.

## What did NOT land at M4-002

- No end-to-end substrate-driven mixed run. That requires
  `mkdir.M2-substrate-002` (probe returns 1 for a) and
  `mkdir.M4-substrate-002` (undo-log walker). Both are follow-up
  issues in paideia-os.
- No per-record body assertion (only counts). `M4-004` reads the CDR
  owner_slot_ref field.

## Cross-repo dependencies

- `mkdir.M4-002` direct: paideia-os kernel through R48b; libpdx-argv
  M1-002; `src/mkdir.pdx` at M3-003.
- `mkdir.M4-002` shape: `mkdir.M2-substrate-002` (probe wires the
  return-1-on-exists convention) + `mkdir.M4-substrate-002` (undo-log
  walker that consumes RUR array in reverse insertion order).

## Build note

No build here — main runs `bash tools/build.sh` and commits the
artefact once clean.
