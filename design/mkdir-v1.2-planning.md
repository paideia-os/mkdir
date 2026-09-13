# mkdir — v1.2.0-A planning: `-p` (parent-creation) + `-m <mode>`

**Status:** PLANNING ONLY — paideia-os/mkdir#33. No implementation in
this commit. Implementation lands in a future `v1.2.0-A` release.

**Basis:** grep-verified against `src/mkdir.pdx`, `deps.list`,
`caps.decl`, `design/user/syscall-table.md`, and
`design/kernel/vfs-metadata-syscalls.md` at HEAD (`v1.4.0` cut). The
current shipped body (v1.1-A → v1.3.0) is a straight-line `_start`
that walks `argv[1..argc]` and calls `sys_mkdir` per positional with a
fixed mode `0755` — **no flag parsing exists today**; `libpdx-argv`
was retired at v1.1-A and is not linked (`deps.list` is empty). Any
`-p` / `-m` design has to either re-adopt `libpdx-argv` or extend the
current inline argv-byte-walk; see §4.

---

## 1. `-p` semantics (idempotent parent-creation)

- **Path decomposition.** Split the operand on `/` into non-empty
  components (mirrors the M2-001 design retired at v1.1-A: `foo//bar`
  → 2 levels, `foo/` → 1, `.` components NOT normalised). Cap at
  `PATH_MAX_COMPONENTS` (16); exceeding it is `MK_PATH_TOO_DEEP`
  (exit 4, matching the existing `MK_PATH_TOO_LONG` convention in
  `src/mkdir.pdx`).
- **Empty-path guard.** An empty operand (`""`) or a path that
  decomposes to zero components yields `comp_count == 0` and
  short-circuits to success (exit 0) — POSIX-idempotent
  `mkdir -p ""` is a no-op, not an error.
- **Per-level walk.** For each component left-to-right, call
  `sys_mkdir` (sysno 79) on the accumulated prefix.
  - `sys_mkdir` returns `0` → level created, continue.
  - `sys_mkdir` returns `-EEXIST` **and** the existing node is a
    directory → treat as success, continue (this is the "existing
    directory OK" idempotence rule). Distinguishing "existing dir" from
    "existing file" needs a `sys_stat` (sysno 77) probe on `-EEXIST`,
    since `sys_mkdir` alone does not report the colliding node's type.
  - `sys_stat` on the `-EEXIST` prefix reports a non-directory node →
    hard error, exit `-EEXIST` (existing-file error, per the issue's
    explicit ask).
- **Fail-fast + undo.** Consistent with the v1.1-A-era
  `mkdir.M3/M4` transaction discipline (now retired from the live
  body but preserved in `design/architecture.md` as prior art): a
  mid-walk failure should not leave a partial mix of newly-created and
  pre-existing levels ambiguous to the caller. Re-adopting a
  `PXT_OP_CREATE` / undo-record scheme is a candidate, but the current
  v1.3.0 body has no `PdxFS TXN` open at all (that substrate was
  dropped along with the M1..M5 stub) — this is an open design
  question for the implementation issue, not resolved here.

## 2. `-m <mode>` semantics

- **Parse octal.** Accept a leading-zero-optional octal string
  (`750`, `0750`, `755`) — three or four octal digits, each `[0-7]`.
  Reject anything else (non-octal digit, empty string, more than 4
  digits, or a value `> 0o7777`) with `MK_INVALID_MODE`, exit 22
  (`EINVAL`), per the issue's explicit ask.
- **Apply after create.** Per the issue text, apply the parsed mode
  "via `sys_chmod`" after a successful `sys_mkdir`.
- **⚠ Blocking kernel dependency — `sys_chmod` does not exist.**
  Grep-verified against the live syscall dispatch
  (`design/user/syscall-table.md`, sysno rows 0..115): there is
  **no `sys_chmod` handler and no `sys_chmod` sysno allocation** in
  paideia-os today. The issue's suggested "sysno 90" is **wrong** —
  sysno 90 is live and already allocated to `sys_accept`
  (`src/kernel/core/syscall/handlers/sys_accept.pdx`, R94.M4-001
  #2068). The "90" figure traces back to a stale, explicitly
  non-binding proposal table in
  `design/kernel/vfs-metadata-syscalls.md` §"informally holds" (dated
  R56.M3-001 planning, marked "None of the above is a hard
  reservation") that was superseded when a later round assigned 86
  to `sys_getcwd` and the socket family claimed 87..90.
  **Implication:** `-m` cannot be implemented against `sys_chmod`
  until paideia-os lands a real `sys_chmod` handler. The next free
  sysno above the current high-water mark (115, `sys_semantic_send`)
  is **116**, subject to the "IDs are negotiable" clause
  (`design/tooling/volume-lifecycle-mechanism.md` §10.1). **Action
  item:** file a paideia-os kernel issue for `sys_chmod` (mode-bits
  write on a `KIND_PDXFS_VOL`-scoped inode) before mkdir's `-m`
  implementation issue is scheduled; mkdir#33's implementation
  follow-up should block on it.
- **Umask interaction.** Per the issue's explicit ask: `-m` **overrides
  umask entirely** (sets exactly the requested bits, not
  `mode & ~umask`). This diverges from POSIX `mkdir -m` (which still
  applies `~umask` to bits not explicitly masked) but matches the
  issue's stated intent and is simpler to implement and reason about
  given paideia-os has no per-task umask state today. Document this
  divergence in `doc/mkdir.pdxdoc` at implementation time.
- **Without `-m`,** the existing fixed mode `0755` (`0x1ED`) stays the
  default — no behaviour change for the common case.

## 3. Interaction between `-p` and `-m`

`-m` applies once, to the **final (leaf) component only** — parent
levels created along the way by `-p` keep the default `0755`. This
matches GNU coreutils `mkdir -p -m`.

## 4. Argv parsing

The issue asks to "extend `libpdx-argv` usage for two new short
flags." Two viable paths, to be decided at implementation time:

1. **Re-link `libpdx-argv`.** Reintroduces a dependency retired at
   v1.1-A (`deps.list` §"Retired from M5-era pins" — grep-verified
   never linked since). Gets `Parser::parse_argv` / well-known-flag
   dispatch for free but reverses a deliberate simplification.
2. **Extend the current inline byte-walk** (`src/mkdir.pdx`'s
   `mkdir_loop_head` positional scan) to recognise `-p` (no operand)
   and `-m <mode>` (consumes the next argv slot) before the positional
   walk, staying dependency-free like v1.1-A through v1.3.0.

**Recommendation:** option 2, for consistency with the shipped
v1.1-A..v1.3.0 design direction (straight-line, dependency-free
`_start`) and because two fixed short flags do not need a general
parser. Final call belongs to the implementation issue.

## 5. Test matrix (placeholder shapes)

- `tests/mkdir_p.pdx`
  - single-level (no `/`) — unchanged from today's behaviour.
  - multi-level, all-new (`a/b/c` with none existing).
  - multi-level, mixed pre-existing + new (`a` exists as a dir,
    `a/b/c` does not) — only `b`, `c` created.
  - multi-level where an intermediate component exists as a **file**,
    not a directory → exit `-EEXIST`.
  - empty-path (`mkdir -p ""`) → exit 0, no syscall issued.
  - `PATH_MAX_COMPONENTS` overflow (17+ levels) → exit 4.
- `tests/mkdir_m.pdx`
  - valid octal (`750`, `0755`) → created, mode verified via
    `sys_stat` once `sys_chmod` lands.
  - invalid mode (`abc`, `9`, `77777`, empty) → exit 22, no
    `sys_mkdir` call attempted (fail before create, so a bad `-m`
    never leaves a directory behind with the wrong/default mode).
  - `-m` + `-p` combined: leaf gets requested mode, parents get
    `0755`.

Both files are **placeholders** at this planning stage — no `.pdx`
test bodies exist yet; they are filed here as the test-matrix shape
the v1.2.0-A implementation issue must satisfy.

## 6. Explicit scope note

This document is planning only (paideia-os/mkdir#33). No `src/`,
`caps.decl`, `deps.list`, or test changes accompany it. Implementation
— including the `sys_chmod` kernel prerequisite in §2 — lands in a
future `v1.2.0-A` release, tracked as a separate implementation issue
against this plan.
