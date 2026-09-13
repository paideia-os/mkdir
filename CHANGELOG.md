# mkdir — CHANGELOG

Semver-tagged release history. Every entry corresponds to a git tag on
this repository and (starting at 1.0.0) a dual-signed
`manifest.pdxsig` at the same commit.

## 1.4.0 — 2026-09-13 — Release closer (fix #31)

Administrative release-closer: this repository accumulated four
stacked `## Unreleased` CHANGELOG headers (Wave E, v1.1-B, v1.1-A,
Enhancement v1.x) across three actual `git tag` cuts (`v1.0.0`,
`v1.2.0`, `v1.3.0`) without the version bookkeeping ever catching
up. Closes paideia-os/mkdir#31 by:

- Finalising the "Unreleased — v1.2.0 (Wave E omnibus)" header below
  into a dated `## 1.2.0` entry matching the existing `v1.2.0` tag
  (commit `ce36a5f`).
- Demoting the v1.1-B / v1.1-A / Enhancement-v1.x headers to `###`
  sub-entries annotated "folded into tag `v1.2.0`" — no standalone
  `v1.1.0` tag was ever cut (confirmed against `git tag`), so those
  three waves ship under the `v1.2.0` release, not a `v1.1.0` one.
- Backfilling a `## 1.3.0` entry (below) for the `mkdir.ENH-002`
  `--json` commit (fix #17, `16bd442`), which landed with no
  CHANGELOG/VERSION/manifest paperwork at all.
- Bumping `VERSION` (`1.2.0` → `1.4.0`), `README.md`'s version
  banner, and regenerating `manifest.pdxsig`'s placeholder file
  inventory and `%ver` field to `1.4.0` (still unsigned —
  `author_pk`/`paideia_root_pk` remain PLACEHOLDER pending the
  v0.33 crypto rollout, T-INFRA-001/002).

**Versioning note.** Issue #31 was filed as "v1.1-C release closer"
against a plan to cut a `v1.1.0` tag. By the time it landed, `v1.2.0`
and `v1.3.0` had already shipped ahead of it in git history. Cutting
`v1.1.0` now, behind two already-released tags, would violate
monotonic tag ordering along mainline, so this closer ships as
`v1.4.0` — the next semver after the current latest tag (`v1.3.0`) —
per the issue's own "or next semver if already exists" fallback,
applied to the de-facto latest tag rather than the literal `v1.1.0`
string.

No source (`src/`) changes in this release.

## 1.3.0 — 2026-09-13 — `--json` stdout output (fix #17)

Backfilled entry — the originating commit (`16bd442`,
`mkdir.ENH-002: --json output on stdout (fix #17)`, already tagged
`v1.3.0`) touched only `src/mkdir.pdx` (149 lines) and shipped with
no CHANGELOG/VERSION/manifest update. See `git show v1.3.0` for the
file-level diff; the `--json` flag serialises the per-invocation
result to stdout instead of (or alongside) the human-readable exit
path. No caps/schema/deps changes.

## 1.2.0 — 2026-09-13 — Wave E omnibus (tag `v1.2.0`)

Bundles the paideia-os/mkdir open-issue drain. Closes five issues in
one signed release cut:

### Closes paideia-os/mkdir#22 — mkdir.ENH-007 real argv (already landed at v1.1-A)

The M5-era `_start` read a hard-coded bootstrap argv literal
(`-p -v --dry-run a/b/c`) so the tool never saw user input. v1.1-A
retired that scaffold and rebuilt `_start` on top of the frozen
execve ABI: rdi = argc, rsi = argv (u64\*), argv[0] = program name.
The `mov rbx, rdi; mov r12, rsi` prologue at `src/mkdir.pdx`
mkdir_loop_head immediate-predecessor is the argv read; every
positional walk (`mov r15, [r12 + r13*8]`) dereferences it directly.

v1.2.0 does not touch that arm. #22 is closed against v1.1-A's
already-landed code path; this CHANGELOG entry cites the closure to
tie the issue to a signed release cut.

### Closes paideia-os/mkdir#28 — R90-XREPO.013.M3-006 caps.decl + adoption

`caps.decl` at v1.1-A was `requires: []` (v1.1-A retired every M5-era
cap-slot placeholder). v1.2.0 lands the R90-XREPO.013.M3-006
adoption profile:

    requires:
      - KIND_PDXFS_VOL(mkdir, <arg-parent>)
      - KIND_IPC_ENDPOINT(write, semantic-pipe)

    declares_output_schemas:
      - MkdirRecord@0.1

The two rows are the real cap footprint for the real body:

- **KIND_PDXFS_VOL(mkdir, \<arg-parent\>)** — the volume-level mkdir-op
  cap, narrowed per positional at exec by shell.M2-001's
  `sys_exec_reconcile_caps` invocation. Cross-subtree targets and
  `/system/` targets are refused at exec (no KIND_ELEVATE_CHANNEL
  declared here; ambient elevate would violate D4).

- **KIND_IPC_ENDPOINT(write, semantic-pipe)** — the ring substrate for
  the v1.2.0 MkdirRecord@0.1 emit. Declared even though the send
  path is best-effort so a downstream broker that enforces "emit or
  refuse" at libpdx-semantic-pipe M3 does not silently drop mkdir's
  records.

The M5-era five-slot declaration (KIND_PDXFS_TXN / KIND_PDXFS_FILE /
KIND_USER + two KIND_IPC_ENDPOINT) is NOT re-added. See
`caps.decl` header for the full rationale.

`manifest.pdxsig [capabilities]` rewritten to the two-row v1.2.0
profile (slot 0 = KIND_PDXFS_VOL(R_PDXFS_VOL_MKDIR); slot 1 =
KIND_IPC_ENDPOINT(R_IPC_WRITE)).

### Closes paideia-os/mkdir#29 — v1.1-A Real-body extraction (already landed)

v1.1-A retired the M1-001 STUB scaffold and rebuilt `_start` on top
of the real `sys_mkdir` syscall (paideia-os sysno 79, kernel body at
`src/kernel/core/syscall/sys_mkdir.pdx`, R56.M3-004 #1793). See the
v1.1-A entry below for the file-by-file inventory. v1.1-B extended
v1.1-A with userspace cwd-relative resolution (paideia-os/mkdir#25);
v1.2.0 extends v1.1-B with the semantic-pipe emit.

The `sys_cap_invoke`-on-placeholder pattern documented in issue #29
was fully retired at v1.1-A; v1.2.0 does not re-add it. #29 is
closed against v1.1-A's already-landed code path.

### Closes paideia-os/mkdir#30 — v1.1-B Semantic-pipe emission wire

Adds the MkdirRecord@0.1 emit via `sys_semantic_send` (SC+ ID 115,
R107-M0-001 #2350; kernel body at
`src/kernel/core/syscall/handlers/sys_semantic_send.pdx`) after
every successful `sys_mkdir`.

#### Record layout (32 bytes total; 4 × u64 lanes)

    [0..8]   path_ptr    -- VA of the NUL-terminated path passed to
                           sys_mkdir. Absolute-path positional: the
                           argv[i] pointer; relative: interior
                           pointer into _mkdir_resolved_scratch.
    [8..16]  path_len    -- byte length of the path (NUL excluded);
                           bounded by SYS_MKDIR_PATH_MAX-1 (255).
    [16..24] mode        -- st_mode of the created directory. Fixed
                           at 0x1ED (0755) at v1.2.0 (no -m flag).
    [24..32] status      -- 0 on success. v1.2.0 emits a record ONLY
                           after sys_mkdir returns 0; failures do
                           not stage a record.

Schema tag: `0x4D6B646972526501` ("MkdirRe\x01") — a magic u64
consumers filter by when they drain `_semantic_ring`.

#### Wire discipline

- Best-effort: sys_semantic_send's return (0 / -EFAULT / -EINVAL)
  is discarded — the directory is already on disk, the 32-byte
  static rip-relative buf never fails user_ptr_ok, and 32 sits
  well inside the [1, 240] length gate. Same discipline pdxsock's
  v1.1-B session-close emit follows (`src/main.pdx` L1917-L1929).

- `_mkdir_record_buf` at `.bss` (@align(8), 32 bytes). mode and
  status are pre-filled once at `_start` entry (post-argc-guard);
  path_ptr and path_len are set per iteration before each
  sys_mkdir call in both the absolute and relative arms.

- Absolute arm gains an inline `strlen(argv[i])` loop (capped at
  255; overflow → exit 4 MK_PATH_TOO_LONG, same exit as the
  relative arm) so the emitted `path_len` matches the byte count
  on the disk-visible path.

#### v1.2.0 files touched by #30

- `src/mkdir.pdx` — new `_mkdir_record_buf` .bss slot; pre-fill
  block after the argc guard; strlen loop in
  `mkdir_use_raw_absolute`; record-field stashes at both the
  absolute arm and the relative arm's post-copy tail;
  sys_semantic_send emit block after the sys_mkdir success check.
- `caps.decl` — `KIND_IPC_ENDPOINT(write, semantic-pipe)` row
  added under `requires:`; `MkdirRecord@0.1` declared under
  `declares_output_schemas:`.
- `manifest.pdxsig [schemas]` — `CreatedDirRecord@0.1` replaced
  by `MkdirRecord@0.1` (the M5-era CreatedDirRecord had no live
  producer/consumer pair; MkdirRecord@0.1 is the v1.2.0 canonical
  shape).

### Closes paideia-os/mkdir#32 — mkdir . refusal smoke

Adds `tests/mk_dot_smoke.pdx` — an in-process
classification-shape-smoke that exercises the exact byte
sequence mkdir's `_start` sees for `mkdir .` (DOT_LITERAL = 0x2E,
0x00) and asserts the classifier's branch decision matches
`mkdir_dot_error` (which would issue sys_exit(1) with the
MK_DOT_CWD_NONSENSICAL diagnostic).

The full runtime harness (`spawn mkdir with argv="."; assert
$? == 1; assert debug-channel emit`) requires shell.M5 (dual-signed
release) + the paideia-os smoke-runner extension; both land AFTER
mkdir v1.2.0. The file's HARNESS RUNBOOK header documents the
paideia-os-side sequence for when those substrates arrive.

Return-code band `0xFFFFEBD0` reserved for MkDotSmoke:

    0            MK_DOT_SMK_OK             all-pass sentinel
    0xFFFFEBD1   MK_DOT_SMK_FIRST_BYTE     first byte != 0x2E
    0xFFFFEBD2   MK_DOT_SMK_SECOND_BYTE    second byte != 0x00

Complementary to `tests/README.md`'s `v1.1-B-T4` note (which
covers the same refusal case as one arm of a broader cwd-relative
smoke); #32's file is a standalone driver that isolates the `.`
classification without a filesystem fixture dependency.

### v1.2.0 tool-ident wire (provisional)

Adds `src/tool_ident.pdx` declaring `PDX_TOOL_NAME : [u8; 6] =
"mkdir\0"`. mkdir does NOT link libpdx-argv at v1.2.0 (deps.list
unchanged from v1.1-A's empty set); the extern is declared
PROVISIONALLY so a v1.3 landing that adds `-m` / `-p` via a
Parser::parse_argv call links against libpdx-argv without a
tool-ident-symbol churn. `PDX_TOOL_VERSION` is intentionally
OMITTED at v1.2.0 — libpdx-argv's `emit_default` needs BOTH
symbols to be safe under `--version`, and mkdir has no
`--version` implementation today; adding the version literal
without a consumer invites doc-truth drift on every bump. Same
posture cat/src/tool_ident.pdx took at v1.2.0-A.

### v1.2.0 files touched (summary)

- `src/mkdir.pdx` — semantic-pipe emit block + record-buf stashes
  (see #30 subsection).
- `src/tool_ident.pdx` — new: PDX_TOOL_NAME extern (provisional).
- `tests/mk_dot_smoke.pdx` — new: `mkdir .` classification smoke
  (see #32 subsection).
- `caps.decl` — full rewrite for R90-XREPO.013.M3-006 adoption
  (see #28 subsection).
- `manifest.pdxsig` — `%ver` bumped 1.0.0 → 1.2.0; `%wave` adds
  R90-XREPO.013.M3-006 + R107-M0-001; `[manifest]` file-hash
  table regenerated to the v1.2.0 inventory (placeholder rows
  awaiting signing-bot rehash); `[capabilities]` narrowed to
  slots 0/1 (KIND_PDXFS_VOL + KIND_IPC_ENDPOINT); `[schemas]`
  swapped CreatedDirRecord@0.1 → MkdirRecord@0.1; `[deps]`
  reconciled to the v1.1-A/v1.2.0 empty set.
- `VERSION` — new: `1.2.0` (satellite version-witness file, joins
  the libpdx-argv precedent).
- `CHANGELOG.md` — this entry.

### v1.2.0 what it does NOT change

- `sys_mkdir` in the kernel is unchanged (v1.1-B's note carries).
- No `-p`, no `-m`, no `-v` — v1.2.0 keeps v1.1-B's flag-free
  surface. `-p` multi-level and `-m <mode>` re-enter at v1.3.
- No shell-side pipe binding (no fd-3 write, no libpdx-semantic-pipe
  fd-bind path). v1.2.0 uses only the direct sys_semantic_send
  ring substrate; the fd-bound producer-broker seam re-enters if
  shell.M3 pipe-plumbing lands with a schema-typed passthrough.
- No `_init_caps` sidecar. mkdir at v1.2.0 continues to declare
  no InitCap array; the shell narrows the ambient cap-set at
  exec per R90-XREPO.013.M2-001 rather than seeding a caller-
  side cap_table.

### v1.1-B (userspace cwd-relative resolution) — folded into tag `v1.2.0`

Closes paideia-os/mkdir#25 (`mkdir.ENH-010 wire relative-path
resolution to the kernel cwd (R86 sys_getcwd)`).

Under v1.1-A mkdir passed each argv positional to `sys_mkdir` raw and
relied on the kernel's `path_resolve(..., cwd=TASK_OFF_CWD)`
(R86.M1-005 #1958) to anchor relatives against the task's TCB cwd.
That was correct once the kernel was updated, but it left the tool
blind to what it was actually creating: the string handed to
`sys_mkdir` was the same "foo" the user typed while the on-disk path
was `getcwd() + "/foo"`. v1.1-B moves the join into userspace so the
string handed to `sys_mkdir` IS the absolute path that lands on disk
— smoke witnesses assert against argv-visible state instead of the
kernel's TCB cwd, and a future backend adapter that does not honour
`TASK_OFF_CWD` (a networked or overlay FS, an out-of-band probe tool)
still gets the correct target because resolution has happened before
the syscall boundary.

### v1.1-B behaviour

Each positional PATH argv[1..argc] is classified by its first byte:

- `'/'` — absolute; passed to `sys_mkdir` as-is (any prefixing would
  be actively wrong since `path_resolve` anchors absolute paths
  against the mount root regardless of cwd).
- `'.'` followed by NUL (exactly `.`) — nonsensical (mkdir the cwd
  itself creates nothing on-disk; the kernel would fail it as
  `-EEXIST` regardless). Emits `mkdir: refusing to mkdir .\n` and
  exits **1** (new: `MK_DOT_CWD_NONSENSICAL`).
- anything else — relative. Calls `sys_getcwd` (SC+ ID 86, R86.M1-003
  #1956 kernel body at `src/kernel/core/syscall/sys_getcwd.pdx`)
  into a 256-byte scratch, then joins `cwd + '/' + path` into a
  second 256-byte scratch and passes that to `sys_mkdir`. The
  separator is elided when cwd is exactly `/` (no double slash on
  `mkdir foo` under root). "./foo", "..", ".foo", and multi-component
  paths like "a/b" all flow through this arm; the kernel's
  `path_resolve` normalises "." components and refuses ".." per its
  own semantics.

### v1.1-B new exit codes

- `1` = `MK_DOT_CWD_NONSENSICAL` — `mkdir .`
- `4` = `MK_PATH_TOO_LONG` — resolved path exceeds `SYS_MKDIR_PATH_MAX-1`
  (255 bytes; cwd_len + sep + user_len must fit with room for the
  trailing NUL at index 255). `argc<2` still exits 2; `-errno`
  passthrough from a failed `sys_mkdir` (or `sys_getcwd`) is
  unchanged from v1.1-A.

### v1.1-B new SC+ IDs

- `86` = `sys_getcwd(buf_va, buf_len) -> u64` — returns bytes written
  INCLUDING the trailing NUL (POSIX `getcwd(3)` convention: N-1 is
  the string's strlen) or negative errno. Inlined at the call site,
  matching the one-file-one-ELF pattern the rest of `_start` uses;
  no `syscall_shim` link dependency.

### v1.1-B new scratches

Two 256-byte `.bss` slots in `Mkdir`:

- `_mkdir_cwd_scratch` — destination for `sys_getcwd`. 256 bytes
  matches `SYS_GETCWD_PATH_MAX` in the kernel's own header, so any
  path the kernel can compose fits without a `-ERANGE` round-trip.
- `_mkdir_resolved_scratch` — staging for the joined absolute path
  handed to `sys_mkdir`. 256 bytes matches `SYS_MKDIR_PATH_MAX` in
  the kernel's own header. Any join exceeding 255 bytes exits `4`.

### v1.1-B corrections to issue #25

Issue #25 references `sys_getcwd` as "sysno 84". That is a typo —
84 is `mountinfo` per `design/user/syscall-table.md` row 84.
`getcwd` is row 86 and every consumer in-tree (`syscall_shim.pdx`
L184, `dispatch.pdx`'s `pwd_builtin` / `cd_builtin`) uses
`mov rax, 86`. v1.1-B follows the syscall table.

Issue #25's "MK_CWD_MISMATCH cross-check" fingerprint targeted the
M5-era `SLOT_PARENT_FILE` (slot 1) cap dispatch shape. v1.1-A
retired every cap slot when it ripped out `_init_caps` and switched
to plain `sys_mkdir` — there is no longer a parent cap for cwd to
mismatch against. v1.1-B addresses the underlying concern (that
mkdir sees an unresolved-against-cwd path) with the more natural
fix for the current shape: perform the resolution explicitly in
userspace so the argv-visible path IS the on-disk path.

### v1.1-B files touched

- `src/mkdir.pdx` — `_start` gains the classify-and-resolve prologue
  before the common `sys_mkdir` call site. Five new diagnostic
  strings (`err_dot_cwd_msg`, `err_getcwd_fail_msg`,
  `err_path_too_long_msg`) and their `_len` companions. Two new
  256-byte `.bss` scratches (`_mkdir_cwd_scratch`,
  `_mkdir_resolved_scratch`).
- `design/architecture.md` — v1.1-B section added ahead of the v1.1-A
  section; the v1.1-A `_start` shape diagram is superseded by a
  v1.1-B version showing the resolve prologue and the shared
  `mkdir_call_sys_mkdir` tail.
- `design/enhancement-plan.md` — `ENH-010` landed entry.
- `doc/mkdir.pdxdoc` — SYNOPSIS unchanged; DESCRIPTION gains the
  "relative paths resolve against `sys_getcwd`" note; EXIT STATUS
  gains rows 1 and 4.
- `tests/README.md` — `v1.1-B-T4` smoke witness added to the
  paideia-os-side matrix note.
- `CHANGELOG.md` — this entry.

### v1.1-B what it does NOT change

- `sys_mkdir` in the kernel is unchanged. `path_resolve`'s
  cwd-anchoring at `sys_mkdir.pdx` L266-272 still fires for any
  future caller that hands it a relative path; v1.1-B just stops
  being one of those callers.
- The tool still refuses no path on validation grounds (v1.1-A's
  behavioural line "does no path validation of its own"). The one
  new refusal is the exact-`.` operand, which is a userspace
  affordance (early diagnostic) rather than a security guarantee.
- No `-p`, no `-m`, no `-v`: v1.1-B keeps v1.1-A's flag-free
  surface. `-p` multi-level with parent creation still relands at
  v1.2-A.

### v1.1-A (real body extraction) — folded into tag `v1.2.0`

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

### Enhancement v1.x (milestone #6) — folded into tag `v1.2.0`

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
