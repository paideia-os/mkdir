# mkdir.M4-004 — implementation notes

**Issue:** #13 — cap-tail correctness (owner matches invoker in every
created inode).
**Upstream doc:** `design/tooling/r49-r50-plan.md` §5.9 (paideia-os).

## What landed

- `tests/test_m4_004_cap_tail.pdx` — `MkdirTestM4004` module with one
  driver `run_test_m4_004_owner`. Invokes `mkdir -p a/b/c` through
  the full parse-run pipeline, then walks the three staged
  `CreatedDirRecord` slots asserting each `owner_slot_ref` == 2
  (SLOT_USER).
- `tests/expected-mkdir-m4-004-owner.txt` — `mkdir.M4-004 owner OK`
- `tests/README.md` + `STATUS.md` — M4-004 promoted to LANDED.

## The M2-003 + M3-001 seam this test freezes

The M2-003 milestone landed the cap-tail stamp at `mkdir_one` Step 6d
— a per-create `sys_cap_invoke(SLOT_USER, USER_OP_QUERY_FP0)` that
exercises the `KIND_USER` slot dispatch. The M3-001 milestone landed
the `CreatedDirRecord` staging at Step 6e with `owner_slot_ref = 2`
(the compile-time constant matching `SLOT_USER`).

M4-004 asserts the seam holds: after `mkdir -p a/b/c` returns MK_OK,
the three staged CDRs (one per level a, b, c) each carry
`owner_slot_ref = 2` at byte offset +24 within their 32-byte record
frame.

The property matters because `ls --long <dir>` (the paideia-os
follow-up) resolves the owner_slot_ref via libpdx-cap M3-001's
`KIND_USER_ref decode` helper — slot 2 -> the invoker's KIND_USER
cap's user_fp_lo. A wrong owner_slot_ref would render the directory
as owned by whatever cap sits at that slot number for the ls process
(possibly a completely unrelated cap or an unbound slot) — a D3/D4
violation because a mkdir'd directory would appear anonymous or owned
by a wrong principal.

## Test shape

Unlike M4-001..M4-003 which assert only on scalar counters, M4-004 is
the first driver that reads CDR record BODIES. The layout freeze from
M3-001:

| word offset | byte offset | field           | value               |
|-------------|-------------|-----------------|---------------------|
| +0          | +0          | path_ptr        | rbx (path base)     |
| +1          | +8          | path_len        | comp_start + comp_len[i] |
| +2          | +16         | parent_txn_id   | current_txn_id      |
| +3          | +24         | owner_slot_ref  | 2 (SLOT_USER)       |

Records are laid out row-major with 4 words (32 bytes) per record.
For record i, the owner_slot_ref lives at `array_base + i*32 + 24`.
The driver forms `i * 32 + 24` as `[base + rcx*8 + 24]` where
`rcx = i * 4` — a shl 2 of the loop counter (an idiom the M3-001
staging code at src/mkdir.pdx line 879 already uses when computing
record_idx * CDR_FIELDS_PER_RECORD).

The walk:
```
    rdx = 0                                ; i
loop_head:
    cmp rdx, 3
    jge done
    mov rcx, rdx
    shl rcx, 2                             ; rcx = i * 4
    lea r11, [rip + created_dir_records]
    mov rax, [r11 + rcx*8 + 24]            ; owner_slot_ref
    cmp rax, 2                             ; SLOT_USER
    jne fail
    add rdx, 1
    jmp loop_head
done:
    ...
```

## Why this test runs at HEAD (no substrate gap)

Both M2-003 and M3-001 already run their bodies at HEAD:

- M2-003's `USER_OP_QUERY_FP0` placeholder always succeeds against
  the pre-seeded `KIND_USER` slot (the kernel-side dispatch at
  `kind_user.pdx` accepts op_id <= USER_OP_MAX = 7).
- M3-001's Step 6e writes `owner_slot_ref = 2` unconditionally from
  the constant `SLOT_USER = 2` in `MkdirState`.

So the natural `mkdir -p a/b/c` run at HEAD stages three CDRs with
owner_slot_ref == 2. M4-004 asserts on that directly — no shape-freeze
workaround needed like M4-002 and M4-003. This test is the tightest
of the four: the substrate seam is already at HEAD, and the assertion
reads the exact byte that libpdx-cap M3-001 will consume when
rendering.

## paideia-as conformance

- Module name PascalCase basename (`MkdirTestM4004`).
- No `test` mnemonic — every zero-check is `cmp reg, 0`.
- `r11` LEA scratch only.
- Every `cmp reg, imm` uses an immediate ≤ 3 (max: loop bound
  `i < 3` and `owner_slot_ref == 2`).
- No push/pop parity — straight-line driver with leaf-return calls at
  the top and a pure-in-.bss walk in the middle (no calls inside the
  loop body).
- Labels prefixed `t4c_` (test-m4-004-cap-tail). Avoids every
  paideia-as reserved keyword.
- Cross-module calls by unqualified linker name.

## What did NOT land at M4-004

- No `parent_txn_id` field body assertion. That's a substrate-side
  invariant (every record shares the same parent_txn_id captured at
  Step 5 via `PXT_OP_QUERY_ID`); the query returns a substrate-specific
  value at HEAD (stub-returned txn_id) that's not stable enough to
  hard-code as an assertion. Substrate follow-up.
- No `path_ptr` / `path_len` field body assertion. That requires
  computing the expected `comp_start_offsets + comp_lengths` for
  every level and comparing — beyond M4-004's scope (which the plan
  doc names as "owner matches invoker in every created inode"). Left
  for `mkdir.M4-substrate-004` if the paideia-os smoke harness wants
  a full per-record body validator.
- No cross-invoker ownership test. Currently only one user cap at
  slot 2; a multi-user test would need shell.M4 + elevate flow.

## Cross-repo dependencies

- `mkdir.M4-004` direct: paideia-os kernel through R48b; libpdx-argv
  M1-002; `src/mkdir.pdx` at M3-003.
- `mkdir.M4-004` shape: libpdx-cap M3-001 (KIND_USER_ref decode
  helper — that consumes owner_slot_ref at ls-render time). Once
  libpdx-cap M3-001 lands, an `ls --long <mkdir'd-dir>` smoke can
  verify end-to-end that the owner rendered matches the invoker's
  KIND_USER fingerprint.

## Build note

No build here — main runs `bash tools/build.sh` and commits the
artefact once clean.
