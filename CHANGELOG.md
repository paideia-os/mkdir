# mkdir — CHANGELOG

Semver-tagged release history. Every entry corresponds to a git tag on
this repository and (starting at 1.0.0) a dual-signed
`manifest.pdxsig` at the same commit.

## Unreleased — v1.1-A (real body extraction)

Retires the M1-001 STUB scaffold: the M1..M5 tree dispatched every
create/probe/stamp/commit step through placeholder `sys_cap_invoke`
ops on four cap slots (KIND_PDXFS_TXN / KIND_PDXFS_FILE / KIND_USER /
KIND_IPC_ENDPOINT) that never actually created a directory on disk,
and read a static bootstrap argv (`-p -v --dry-run a/b/c`) so the
tool could not see what a user typed. v1.1-A rebuilds `_start` on
top of the real `sys_mkdir` syscall (paideia-os sysno 79, kernel
body at `src/kernel/core/syscall/sys_mkdir.pdx`, R56.M3-004 #1793)
and the frozen execve ABI's own-aspace argv wire (rdi=argc,
rsi=argv). The result is the minimum viable tool a paideia-os user
can invoke and observe a directory afterwards.

### v1.1-A behaviour

    mkdir <PATH> [<PATH>...]

- Each positional is passed to `sys_mkdir` with mode 0755 (0x1ED)
  and a `path_len_hint` of 255 -- the walker CAP convention per
  paideia-os `design/user/syscall-table.md` §Walker length hint
  semantics.
- Exit 0 on all-success; on the first `sys_mkdir` that returns
  non-zero the tool exits with that value (a negative errno per
  the paideia-os -errno sentinel convention, passed through to
  `sys_exit`'s status word verbatim).
- `argc < 2` (no positional) emits `mkdir: missing operand\n` to
  the debug channel and exits 2.
- No `-p`, no `-m`, no `-v`, no `--dry-run`, no `--schema`, no
  `--version`: v1.1-A recognises no flags. `--help` remains
  shell-dispatched to `doc mkdir`.

### What v1.1-A retires

- `src/mkdir_state.pdx` deleted. v1.1-A carries no state across
  loop iterations, so `MkdirState`'s `.bss` singletons, per-
  component bookkeeping arrays, and `CreatedDirRecord[]` /
  `RmdirUndoRecord[]` staging are all unnecessary.
- `_init_caps` sidecar removed from `src/mkdir.pdx`. `sys_mkdir` is
  a plain `@{fs}` syscall; it does not thread through the
  capability subsystem, so v1.1-A takes zero caps at exec (matching
  every `/bin` coreutil in the paideia-os user tree).
- `parse_flags_from_argv`, `mkdir_split_path`, `mkdir_one`,
  `mkdir_run`, `emit_audit_event`, `emit_stderr`,
  `mkdir_state_reset` all deleted. `_start` is one straight-line
  loop.
- `libpdx-argv`, `libpdx-cap`, `libpdx-audit`,
  `libpdx-semantic-pipe` dependencies dropped from `deps.list`.
  `libpdx-argv` was the one M5-era pin that was actually linked;
  v1.1-A reads argv directly. The other three had been grep-verified
  stale since `mkdir.ENH-009` (see `design/enhancement-plan.md`
  §3.7); v1.1-A completes their retirement.
- `caps.decl` narrowed to `requires: []` +
  `declares_output_schemas: []`. `CreatedDirRecord@0.1` re-enters
  as a live schema when v1.2-A relands the emit against a real
  `libpdx-semantic-pipe` consumer (grep-verified zero call sites
  in the ecosystem at v1.1-A).
- `tests/` M4 + ENH-012 drivers and their `expected-*.txt` witnesses
  all deleted -- every driver asserted against the retired
  `MkdirState` singletons. `tests/README.md` records the v1.1-A
  smoke-witness follow-ups (v1.1-A-T1..T3) that belong in the
  paideia-os smoke matrix rather than this repo's in-process
  driver shape (v1.1-A no longer exposes a `mkdir_run`-style
  function for a driver to call into).

### What v1.1-A defers

- `-p` multi-level with parent creation. v1.2-A relands the
  `mkdir_split_path` walker on top of the real substrate --
  because v1.1-A's create primitive is now real, the walker no
  longer needs the M2-era pre-existence-probe / cap-tail-stamp
  placeholders; it becomes a straight per-component `sys_mkdir`
  with pre-existing hits absorbed via `sys_stat` (`-ENOENT` vs
  hit).
- `-m <mode>` custom mode. v1.2-A adds argv-consumed mode parsing
  once the flag-recogniser reappears with `-p`.
- Audit-journal emit (`UEJ_KIND_TOOL_INVOKE` /
  `UEJ_KIND_TOOL_ERROR`). v1.2-A relands the emit on top of the
  real `libpdx-audit` marshaller (the M5-era hand-rolled
  `sys_cap_invoke` shape was never a real library call).
- `CreatedDirRecord@0.1` semantic-pipe emit. Relands together with
  the audit hookup at v1.2-A once a real `libpdx-semantic-pipe
  send_record` consumer exists.

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
- **ENH-006 (#21)** — hoisted TXN open + commit out of `mkdir_one`
  (called once per positional) into `mkdir_run` (called once per
  invocation), fixing two bugs in `mkdir a b c`: every positional past
  the first used to issue its own `PXT_OP_COMMIT` against the
  already-COMMITTED shared TXN row and fail `MK_TXN_COMMIT_FAIL`
  (double commit), and `newly_created_count` was reset to 0 at the top
  of every `mkdir_one` call, so each positional's `CreatedDirRecord` /
  `RmdirUndoRecord` overwrote the previous one at index 0 (record
  clobber). `mkdir_run` now opens one TXN, calls `mkdir_one` per
  positional (records now append via a running `newly_created_count`),
  and issues one commit only after every positional succeeds; a
  failure anywhere leaves the shared TXN uncommitted.
- **ENH-001 (#16)** — implemented `--schema`: `parse_flags_from_argv`
  recognises the `schema` long flag; `_start` emits the
  `CreatedDirRecord@0.1` field list and exits 0 before the
  missing-operand check and before any TXN cap invocation (so
  `mkdir --schema` with no path operand succeeds).
- **ENH-003 (#18)** — implemented `--version`: emits
  `mkdir 1.0.0 (build unknown) sig=placeholder` and exits 0, same
  before-missing-operand / no-cap-invocation placement as `--schema`.
  `build`/`sig` are static placeholders — no build-time hash/signature
  substitution mechanism exists in this tree yet. Also corrected
  `doc/mkdir.pdxdoc` and `CHANGELOG.md`'s 1.0.0 entry: `--help` is
  (and always was) shell-dispatched to `doc mkdir`, not parsed by
  mkdir itself; the SYNOPSIS and FLAGS sections now say so plainly.
- **ENH-008 (#23)** — `caps.decl`'s `requires:` list was missing slot 4
  (the `KIND_IPC_ENDPOINT` audit-journal cap `emit_audit_event` invokes
  on every invocation) even though `_init_caps` (src/mkdir.pdx) and
  `manifest.pdxsig [capabilities]` both already listed it; added, now
  byte-consistent across all three. Also corrected a false claim
  repeated in `caps.decl`, `doc/mkdir.pdxdoc`, and
  `design/architecture.md`: `cap_manifest_verify` does NOT run at
  mkdir's `_start` (grep-confirmed: zero call sites), and no confirmed
  mechanism anywhere in the ecosystem currently supplies the "received
  Cap[] wire array" argument it would need for a spawned coreutil.
  mkdir's actual cap-delivery enforcement is the kernel loader's
  InitCap validator (`init_caps_validate`), which runs against
  `_init_caps` at image-load time — the docs now say so.
- **ENH-011 (#26)** — implemented `-v`: `mkdir_one` Step 6g now emits
  `mkdir: created <path>` per created level via three `emit_stderr`
  calls (static prefix, the same cumulative-prefix `path_ptr`/`path_len`
  the `CreatedDirRecord`/`RmdirUndoRecord` staging computes, static
  newline). Pre-existing levels under `-p` and `--dry-run` invocations
  emit nothing, matching `doc/mkdir.pdxdoc`'s existing (previously
  false) description.
- **ENH-009 (#24)** — reconciled `deps.list` to actual linkage: dropped
  `libpdx-cap`, `libpdx-audit`, and `libpdx-semantic-pipe` (grep-verified
  zero call sites — only `libpdx-argv`'s `reset`/`parse_argv` are
  actually called), each recorded in the "v1.0 does NOT depend on"
  block with its specific reason and re-add condition. `pkg install`
  no longer makes mkdir's installability depend on three unused
  dual-sig verifications. `manifest.pdxsig [deps]` still lists all
  four — that section is part of the signed manifest body (frozen at
  v1.0 per plan.md §D4) and is reconciled at the next signed manifest
  revision, not hand-edited here.
- **ENH-012 (#27)** — extended the test matrix with the three shapes
  the M4 matrix never exercised: `test_enh_multi_positional.pdx`
  (`mkdir a b c` — one shared TXN + per-positional record fields, the
  mkdir.ENH-006 regression test), `test_enh_record_fields.pdx`
  (`mkdir a` asserting `created_dir_records[0].path_len == 1` not 0 —
  mkdir.ENH-005 — plus `mkdir -p a/b/c` asserting per-level `path_len`
  1/3/5 against one shared `path_ptr`), and `test_enh_path_guards.pdx`
  (`mkdir -p ../x` and `mkdir -p a/../../x`, both asserting
  `MK_PARENT_REF_UNSUPPORTED` — mkdir.ENH-004). Every prior M4 driver
  asserted record COUNTS only, never a record FIELD, and used exactly
  one positional — that gap is why all three defects shipped inside a
  green M4.

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

  **Correction (added post-tag, see Unreleased above):** none of
  `--version`, `--schema`, or `--json` were actually implemented in
  `src/` at this tag — `parse_flags_from_argv` recognised only `p`,
  `v`, `dry-run`. This section recorded the plan-of-record at release
  time, not verified working code; `--help` was (and remains) correctly
  shell-dispatched. `--schema` and `--version` were implemented for
  real at mkdir.ENH-001 / mkdir.ENH-003 (Unreleased); `--json` remains
  open as mkdir.ENH-002.

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
