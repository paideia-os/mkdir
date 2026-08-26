# mkdir — CHANGELOG

Semver-tagged release history. Every entry corresponds to a git tag on
this repository and (starting at 1.0.0) a dual-signed
`manifest.pdxsig` at the same commit.

## Unreleased — Enhancement v1.x (milestone #6)

Post-1.0 correctness + doc-truth fixes tracked in
`design/enhancement-plan.md`. Landing incrementally; each entry below
corresponds to one closed `mkdir.ENH-NNN` issue.

- **ENH-004 (#19)** — subtree containment guard: `mkdir_split_path`
  rejects any path component of exactly two bytes ".." with the new
  `MK_PARENT_REF_UNSUPPORTED` (10) return code, checked before any cap
  invocation. Closes the gap where the leading-'/' guard alone did not
  stop `mkdir -p ../../etc` from walking outside the invoker's subtree.
- **ENH-005 (#20)** — the non-`-p` single-component path
  (`mkdir_one_no_slash`) now populates `comp_start_offsets[0]` /
  `comp_lengths[0]` instead of leaving them uninitialized. Previously
  every `CreatedDirRecord` / `RmdirUndoRecord` staged by a plain
  `mkdir a` carried `path_len == 0` (unrenderable by `ls --long`,
  unremovable by `undo`).

## 1.0.0 — 2026-08-22 — first signed release

Wave R50. Milestone M5 close: `mkdir` is *released* per the
`design/tooling/r49-r50-plan.md` §5.9 rubric.

### Release contents

- `src/mkdir.pdx`, `src/mkdir_state.pdx` — the two-module Mkdir tool
  built at M1..M3.
- `tests/test_m4_00[1-4]_*.pdx` + `expected-*.txt` — the M4 test
  matrix landed at commits `fd4797e`..`b12b264`.
- `caps.decl`, `deps.list`, `doc/mkdir.pdxdoc`, `manifest.pdxsig` —
  the M5-001 release scaffolding.
- `design/architecture.md` — internal design frozen at v1.0.

### What landed at each milestone

- **M1 (#1, #2, #3)** — scaffold + argv wire-up + single-level
  `mkdir a` in a KIND_PDXFS_TXN with the invoker cap as owner.
- **M2 (#4, #5, #6)** — `-p` multi-level atomic in one TXN, pre-existing
  levels no-op, cap-tail owner stamp on every created inode.
- **M3 (#7, #8, #9)** — `CreatedDirRecord[]` schema on the semantic
  pipe, `CreateDirRecord` via libpdx-audit, `RmdirUndoRecord[]` PdxFS v1
  undo record.
- **M4 (#10, #11, #12, #13)** — single/multi test, mixed pre-existing
  under -p (undo scope), TXN-abort mid-create (no dirs left),
  cap-tail correctness per created inode.
- **M5 (#14, #15)** — dual-signed manifest.pdxsig + doc/mkdir.pdxdoc +
  CHANGELOG-1.0 (this entry) + deps.list; mirror-push staging manifest
  landed at `.pkgs-mirror/staging.pdxpush` per §6.3.

### Cross-repo dependency versions pinned

    libpdx-argv@1.0.0
    libpdx-cap@1.0.0
    libpdx-audit@1.0.0
    libpdx-semantic-pipe@1.0.0

libpdx-elevate is intentionally NOT a v1.0 dependency — mkdir writes
only inside the invoker's own PdxFS subtree at v1.0 (§5.9 "no elevate
dependency" line).

### Placeholder-substrate seams still open at v1.0

`mkdir.M2-substrate-{001,002,003}` at the paideia-os repo are open;
the tool ships against those seams shape-frozen — Step 6a
(pre-existence probe), Step 6b (create), Step 6c
(newly_created_count), Step 6d (owner stamp), and Step 7 (commit) all
carry M2/M3-era placeholders that surface the cap dispatch shape but
call query ops instead of the future write ops. When the substrate
lands, the bodies are edited without a signature change to `mkdir_one`.

### Diagnostics

- `mkdir --version` → `mkdir 1.0.0 (build b12b264) sig=<placeholder>`
  (real fingerprint replaces the placeholder at M5-002 signing-bot pass).
- `mkdir --help` → shell dispatches to `doc mkdir` which renders
  `doc/mkdir.pdxdoc`.
- `mkdir --schema` → emits `CreatedDirRecord@0.1` on stdout and exits 0.

### Signing pipeline status

- author_pk: **placeholder** — real key generated at T-INFRA-002 in
  `design/tooling/plan.md` §10 (paideia-os).
- paideia_root_pk: **placeholder** — real key generated at T-INFRA-002.
- The signing bot (T-INFRA-002 host) will replace both signature blocks
  in `manifest.pdxsig` without editing the manifest body — the byte
  range under signature is frozen at v1.0.

### Known limitations (v1.0)

- Absolute paths refused (`MK_ABS_PATH_UNSUPPORTED`). Cross-subtree
  writes wait on libpdx-elevate hop (post-1.0 minor bump).
- No `-m <mode>` flag. Access is cap-mediated via KIND_PDXFS_FILE
  rights, not POSIX mode bits. A `--caps <caps.decl>` alternative
  is out of scope for v1.0.
- Dot components (`foo/./bar`) propagate to the placeholder create
  as level `.` — kernel-side normalisation is `mkdir.M2-substrate-001`.
- Parent-reference (`..`) components are rejected as of the mkdir.ENH-004
  fix: `mkdir_split_path` returns `MK_PARENT_REF_UNSUPPORTED` (10) for
  any component of exactly two bytes "..", before any cap invocation.
  This closes the subtree-containment gap the leading-'/' guard alone
  did not cover (`mkdir -p ../../etc` previously split cleanly and
  would have walked outside the invoker's subtree once
  `mkdir.M2-substrate-001` lands the real create-dir walker body).
- PATH_MAX_COMPONENTS = 16 (§4a of `design/architecture.md`); deeper
  paths return `MK_PATH_TOO_DEEP`. Sufficient for every realistic
  paideia-os path (deepest at R48 is 4 levels).
