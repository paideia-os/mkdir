# mkdir — enhancement plan (post-v1.0.0)

**Wave:** R50 coreutil → Enhancement v1.x
**Repo:** github.com/paideia-os/mkdir
**Milestone:** #6 "Enhancement v1.x — mkdir"
**Basis:** every claim below was grep-verified against `src/mkdir.pdx`,
`src/mkdir_state.pdx`, `caps.decl`, `deps.list`, `manifest.pdxsig`, and
`tests/` at commit `8eda3e0`. Nothing here is inferred from prose.

This document lives in `design/` rather than a new `docs/` tree because
the repo already splits internal specs (`design/architecture.md`) from
the user-facing man page (`doc/mkdir.pdxdoc`); an enhancement plan is an
internal spec.

---

## 1. Current state

`mkdir` is two paideia-as modules — `MkdirState` (constants, `.bss`
staging, `mkdir_state_reset`) and `Mkdir` (`_init_caps`, `_start`,
`parse_flags_from_argv`, `mkdir_split_path`, `mkdir_one`, `mkdir_run`,
`emit_stderr`, `emit_audit_event`). M1–M5 all landed; tag `v1.0.0` sits
at the M5-001 commit.

What genuinely works at HEAD:

- `-p` multi-level splitting (`mkdir_split_path`) with empty-component
  absorption, a 16-level bound, and a leading-`/` refusal.
- A per-component walk inside one TXN scope with a pre-existence probe,
  `newly_created_count` bookkeeping gated on actual creates, and a
  cap-tail owner stamp per created level.
- `CreatedDirRecord@0.1` and `RmdirUndoRecord@0.1` staged into `.bss`,
  with both counts frozen before the commit syscall so a commit failure
  leaves a consistent "staged, never committed" snapshot.
- D3 audit-first: `UEJ_KIND_TOOL_INVOKE` is emitted before any argv
  work; `UEJ_KIND_TOOL_ERROR` before every non-zero exit; both returns
  ignored so a saturated journal cannot sink the op.

The shape is genuinely good. The gaps are that almost none of it is
reachable by a real user, and several documented behaviours do not
exist.

---

## 2. The `--json` / `--schema` doc-vs-source gap

### 2.1 What the docs claim

- `doc/mkdir.pdxdoc` L14 SYNOPSIS lists `--json`, `--schema`, `--help`,
  `--version`; the FLAGS section gives each a full description, calling
  `--json` and `--schema` "I3 cross-tool contract".
- `CHANGELOG.md` "Diagnostics" asserts `mkdir --schema` "emits
  `CreatedDirRecord@0.1` on stdout and exits 0" and that `mkdir
  --version` prints `mkdir 1.0.0 (build b12b264) sig=<placeholder>`.

### 2.2 What the source does

`parse_flags_from_argv` byte-matches exactly three names — `p`, `v`,
`dry-run`. A grep for `json|schema|help|version` across `src/` returns
only comments and the `CDR_SCHEMA_HANDLE` constant. There is no
recogniser, no emitter, no exit path for any of the four flags. They are
parsed by libpdx-argv into `flag_names[]` and then silently discarded.

`README.md` (L69–71) is already honest about this. `doc/mkdir.pdxdoc`
and `CHANGELOG.md` are not.

### 2.3 Decision — implement `--schema` and `--json`, keep `--version`,
### shell-dispatch `--help`

**`--schema`: implement.** This is not copy-pasted cruft. mkdir already
declares `CreatedDirRecord@0.1` under `declares_output_schemas` in
`caps.decl`, already carries a frozen `CDR_SCHEMA_HANDLE =
0x0100000100010001`, and already stages fully-formed records. `--schema`
is a static string emit plus `exit 0` — the cheapest possible thing, and
it is the flag that makes the tool *introspectable* rather than merely
*structured*. A semantically-queryable terminal in which a consumer
cannot ask a tool what shape it emits, without running it, is not
queryable. XS effort, direct pillar payoff.

**`--json`: implement, but sequenced behind the correctness fixes.**
The stronger argument for stripping it is that I4 makes the schema
stream the primary output and text diagnostic-only, so re-serialising
records as JSON on stdout looks like a POSIX-world affordance in tension
with the pillar. That argument loses on one fact: **cap slot 3, the
`KIND_IPC_ENDPOINT` semantic pipe, is declared in `_init_caps`, declared
in `caps.decl`, declared in `manifest.pdxsig`, and never invoked once.**
The records mkdir builds have no reader. `--json` and the slot-3 pipe
emit are the same missing last mile, and the pipe is the part that
actually matters; JSON is the escape hatch for consumers outside the
pipe. Implementing them together turns a dead capability into a live
one. Sequenced after the `path_len` fix (§3.2), because emitting records
whose `path_len` is 0 would ship the bug to every consumer.

**`--version`: implement.** Twenty lines of static emit; the CHANGELOG
already promises users a specific output string.

**`--help`: do not add a mkdir-side recogniser.** The `.pdxdoc` itself
says `--help` is "hyperlinked into `doc mkdir`", i.e. the shell
dispatches it and mkdir never sees it. That is the correct design — 14
tools should not each inline a help renderer. The work here is doc
truth, not code: state plainly that `--help` is shell-dispatched.

This is deliberately not "implement everything the doc promised". One of
the four flags is being resolved by correcting the document instead.

---

## 3. Verified defects (source-confirmed, not inferred)

### 3.1 `..` components are unguarded — containment escape

`mkdir_split_path` refuses a leading `/` and skips empty components, but
treats `..` as an ordinary component. `mkdir -p ../../etc` splits
cleanly into `[.., .., etc]` and the walk creates/traverses upward.

Every doc in this repo — `caps.decl` header, `.pdxdoc` DESCRIPTION,
`README.md` — states that v1.0 writes only inside the invoker's own
subtree and that this is why there is no libpdx-elevate dependency. The
leading-`/` guard is the only thing enforcing that claim, and `..`
walks straight around it. `CHANGELOG.md` "Known limitations" mentions
`.` propagating as a level named `.`, and never mentions `..` at all.

Today the blast radius is bounded because `PXT_OP_CREATE` is a stub. The
moment `mkdir.M2-substrate-001` lands the real walker body, this becomes
a live subtree escape. **Highest-severity item in this plan.**

### 3.2 Non-`-p` creates stage records with `path_len == 0`

`mkdir_one_no_slash` sets `comp_count = 1` and jumps to
`mkdir_one_after_split` — it never writes `comp_start_offsets[0]` or
`comp_lengths[0]`. Those arrays are `uninit @align(8)`, and
`mkdir_state_reset` deliberately does not zero them (consume-by-index
discipline). Step 6e/6f then compute:

    path_len = comp_start_offsets[walk_i] + comp_lengths[walk_i]

For a plain `mkdir a`, that is `0 + 0 = 0`. So the `CreatedDirRecord`
and the `RmdirUndoRecord` for the single most common invocation of the
tool both name a zero-length path: `ls --long` cannot render it, and
`undo` cannot remove it.

No test catches this — a grep for `path_len` across `tests/` returns one
comment and zero assertions. `test_m4_001_single_multi.pdx` runs exactly
this scenario (`argv_single_argc = 1`, `argv[0] = "a"`) and asserts only
the four counters.

### 3.3 Multi-positional invocation is broken three ways

The synopsis in `README.md`, `.pdxdoc`, and `architecture.md` all
advertise `mkdir [-p] [-v] [--dry-run] <path> [<path>...]`.
`mkdir_run` does walk `pos_ptrs[0..pos_count]`. But `mkdir_one` is not
re-entrant against the shared TXN and the shared record arrays:

1. **Record clobber.** `mkdir_one` zeroes `newly_created_count` at Step
   6 entry, and records are indexed at `newly_created_count - 1`. For
   `mkdir a b`, path `b`'s records overwrite path `a`'s at index 0, and
   both counts freeze to the last path's total. Every path but the last
   loses its undo record.
2. **Double commit.** Each `mkdir_one` issues `PXT_OP_COMMIT` on the
   same pre-opened slot-0 TXN. The second positional hits an
   already-COMMITTED row → `PXT_BAD_TRANSITION` (negative) →
   `MK_TXN_COMMIT_FAIL`. `mkdir_state.pdx` L86–88 explicitly reasons
   that "mkdir is a one-shot process per `_start` invocation, so the
   second-commit edge is unreachable at v1.0" — true per *process*,
   false per *positional*.
3. **Create-after-commit.** For the same reason, path 2+'s
   `PXT_OP_CREATE` targets a committed TXN row.

Untested: every M4 driver uses exactly one positional (`argc` of 1 or 2,
the 2 being `-p` plus one path).

The architecturally correct fix is to hoist TXN open and commit out of
`mkdir_one` into `mkdir_run`, so one TXN spans the whole invocation and
records append across positionals — which is also what "atomic" ought to
mean for `mkdir a b c`.

### 3.4 `_start` ignores real argv

`_start` Step 3 builds a static four-entry array over `.rodata`:
`-p`, `-v`, `--dry-run`, `a/b/c`. There is no read of a loader-supplied
argv anywhere. The shipped v1.0.0 binary therefore cannot create a
directory a user names — it always dry-runs `a/b/c` and exits 0.

`README.md` is honest about this ("the real argv-in-sidecar wire waits
on shell.M4"). It remains the single largest gap between the tool and
what a user at HEAD needs.

### 3.5 Relative paths resolve against a spawn-time cap, not the kernel cwd

Grep for `getcwd|chdir|cwd` across the entire repo: zero hits. mkdir
accepts only relative paths (absolute ones are refused) and resolves
them implicitly against whatever parent inode the shell seeded into
`SLOT_PARENT_FILE` (slot 1) at spawn time.

R86 landed real kernel cwd (`sys_chdir` / `sys_getcwd`) *after* mkdir
shipped v1.0.0. There are now two divergent notions of "where relative
paths land": the kernel cwd, and mkdir's spawn-time parent cap. Nothing
in mkdir asserts, verifies, or reconciles them. `cd /foo; mkdir bar`
creates `bar` under the spawn-time parent cap, not under the kernel cwd,
unless the shell re-seeds slot 1 on every `cd` — an obligation that is
written down nowhere.

Two defensible resolutions: resolve against `sys_getcwd` directly, or
keep the parent-cap contract and make it explicit and asserted. This
plan picks the latter as primary (it preserves least-authority: mkdir
gets a cap to one directory, not ambient reach) with an explicit
`sys_getcwd` cross-check so a divergence is a loud error rather than a
silent write to the wrong place.

### 3.6 Capability manifest is under-declared and unverified

- `_init_caps_count = 5` and `_init_caps` entry 4 is the
  `svc.audit-journal` `KIND_IPC_ENDPOINT`. `manifest.pdxsig`
  `[capabilities]` lists five rows. **`caps.decl requires:` lists only
  slots 0–3.** The audit cap mkdir actually uses on every single
  invocation is absent from the declaration.
- `manifest.pdxsig` says its capability block is "copied verbatim from
  caps.decl". It is not — it has a row caps.decl lacks.
- `caps.decl` L16–17 and `.pdxdoc` L86–87 both state that
  `cap_manifest_verify` runs at `_start` and refuses any cap not
  declared. It does not run: grep finds `cap_manifest_verify` in exactly
  one place repo-wide, a comment in `src/mkdir.pdx` L221.

The compound failure mode is sharp: if `cap_manifest_verify` is made
strict as promised *before* caps.decl gains slot 4, the audit endpoint
is refused and mkdir's D3 audit-first commitment silently dies.

### 3.7 deps.list overstates linkage by three of four libraries

`deps.list` pins `libpdx-argv`, `libpdx-cap`, `libpdx-audit`, and
`libpdx-semantic-pipe`, all at 1.0.0. Actual cross-module calls in
`src/`: `reset` and `parse_argv`, both libpdx-argv. That is all.

- `libpdx-cap` — `cap_manifest_verify` is never called.
- `libpdx-audit` — `emit_audit_event` is a hand-rolled raw
  `sys_cap_invoke`, not a library call.
- `libpdx-semantic-pipe` — `send_record` is never called; slot 3 is
  never invoked.

Per deps.list's own header, `pkg install mkdir` resolves this list
transitively and refuses the install if any row's dual-sig verification
fails. mkdir therefore makes its own installability depend on three
libraries it does not use — pure supply-chain surface for zero benefit.

Correct direction: land the real `cap_manifest_verify` and
`send_record` calls (§3.6, §2.3) so two of the three pins become true,
and drop `libpdx-audit` until `emit_audit_event` actually routes through
it.

### 3.8 Smaller items

- **`-v` emits nothing.** `.pdxdoc` promises `mkdir: created <path>`
  per level; Step 3 in `mkdir_one` is a comment. Cheap once §3.2 lands,
  since the per-level `path_len` becomes correct.
- **`SLOT_STDERR : u64 = 3`** in `mkdir_state.pdx` L41 is a dead
  constant whose name contradicts caps.decl's slot 3 = semantic pipe.
- **`.` components** propagate as a level literally named `.`
  (documented, deferred to substrate — genuinely lower priority than
  `..`, which is not documented at all).

---

## 4. Gap vs what a paideia-os user needs at HEAD

Ranked by what stands between the tool and usefulness:

1. It cannot read argv (§3.4) — nothing else matters until this lands.
2. Relative paths may land somewhere other than the user's cwd (§3.5).
3. `mkdir a b` fails and loses undo records (§3.3).
4. `mkdir a` stages an unusable undo record (§3.2).
5. `..` escapes the subtree the docs promise it cannot (§3.1) — lower
   user-visibility, highest severity once the substrate lands.
6. The records it builds have no reader (§2.3, slot 3 dead).
7. Documented flags don't exist (§2).

Items 1 and the kernel-side halves of 2 and 6 have paideia-os monorepo
companions; see §6.

---

## 5. Issue plan

Filed into milestone #6, titled `mkdir.ENH-NNN`.

| ID | Issue | Title | Effort | Deps | §  |
|----|-------|-------|--------|------|----|
| ENH-001 | #16 | `--schema` static emit | XS | none | 2.3 |
| ENH-002 | #17 | `--json` + live slot-3 semantic-pipe emit | M | #16, #20 | 2.3 |
| ENH-003 | #18 | `--version` emit; correct `--help` doc claim | S | none | 2.3 |
| ENH-004 | #19 | Reject `..` in `mkdir_split_path` | S | none | 3.1 |
| ENH-005 | #20 | Populate component arrays on the non-`-p` path | S | none | 3.2 |
| ENH-006 | #21 | Hoist TXN to `mkdir_run`; append records across positionals | L | #20 | 3.3 |
| ENH-007 | #22 | Read real argv; drop the static bootstrap | L | none | 3.4 |
| ENH-008 | #23 | Declare slot 4 in caps.decl; call `cap_manifest_verify` | M | none | 3.6 |
| ENH-009 | #24 | Reconcile deps.list to actual linkage | XS | #17, #23 | 3.7 |
| ENH-010 | #25 | Relative-path resolution against the kernel cwd | M | none | 3.5 |
| ENH-011 | #26 | `-v` per-level emission | S | #20 | 3.8 |
| ENH-012 | #27 | Test matrix: multi-positional, `path_len`, `..`, non-`-p` | M | #19, #20, #21 | — |

Suggested landing order: 005 → 004 → 001 → 003 → 008 → 006 → 007 → 010
→ 002 → 011 → 012 → 009. The cheap correctness fixes (005, 004) unblock
everything downstream and are each a handful of instructions; 009 lands
last because 002 and 008 are what make two of its three stale pins true.

---

## 6. paideia-os monorepo companions (flagged, not filed here)

A coordinating pass consolidates these; they are recorded so the
dependency is not lost:

- **argv-in-sidecar wire** (`shell.M4`) — ENH-007 needs a stable
  loader/shell argv convention to read.
- **`mkdir.M2-substrate-{001,002,003}`** — the create / existence-probe
  / owner-stamp walker bodies. Still open; ENH-004's `..` guard should
  land *before* substrate-001, since substrate-001 is what makes the
  escape live.
- **Shell must re-seed `SLOT_PARENT_FILE` from the live kernel cwd on
  every `cd`** (§3.5), or publish the base-cap convention that lets
  mkdir resolve against `sys_getcwd` itself.
- **`libpdx-semantic-pipe` `send_record`** must be callable for ENH-002.
- **`bin_seeds`** — mkdir needs a `/bin/mkdir` seed entry to be
  invocable at all from the shell.

---

## 7. Verdict on the `v1.0.0` tag

The tag is fine as an immutable snapshot of the M5-001 commit, and the
engineering underneath it is careful — the staging discipline, the
count-freeze-before-commit ordering, and the audit-first sequencing are
all genuinely well-reasoned.

But `STATUS.md`'s claim that mkdir is "*released* per the
`r49-r50-plan.md` §5.9 rubric" overstates it. A released coreutil that
cannot read its own argv, whose documented multi-path synopsis fails on
the second path, and whose CHANGELOG asserts a `--schema` output that
has no implementation, is shape-frozen scaffolding with a release tag on
it. The README is admirably honest; the CHANGELOG and `.pdxdoc` are not,
and that asymmetry is the part worth fixing first.

Recommendation: leave `v1.0.0` in place, and cut a `1.0.1` doc-truth
release (ENH-003 plus the `.pdxdoc` / CHANGELOG corrections) ahead of
the functional work, so no user is reading a promise the binary does not
keep.
