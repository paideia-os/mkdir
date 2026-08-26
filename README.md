# mkdir

paideia-os directory create — a R50 coreutil that creates one or more
directories inside a single `KIND_PDXFS_TXN`, stamps the invoker's
`KIND_USER` cap as the owner tail of every created inode, and journals
the invocation.

## Synopsis

    mkdir [-p] [-v] [--dry-run] <path> [<path>...]

Install:

    $ pkg install mkdir

`pkg` verifies the dual-signed `manifest.pdxsig` (ML-DSA-65 under
`author_pk` + `paideia_root_pk`) before extracting into
`/pkgs/mkdir-1.0.0/` and symlinking `/bin/mkdir`.

## Description

`_start` (`src/mkdir.pdx`) runs a fixed sequence: ring3 marker →
`mkdir_state_reset` → libpdx-argv `reset` → **audit `TOOL_INVOKE` emit**
→ argv build → `parse_argv` → `parse_flags_from_argv` → operand check →
`mkdir_run` → conditional audit `TOOL_ERROR` emit → `sys_exit(exit_code)`.
The `TOOL_INVOKE` record is emitted *before* any argv work, so a tool
cannot suppress its own audit trail (D3 audit-first).

`mkdir_run` owns the TXN scope for the **whole invocation** (mkdir.ENH-006):
it opens the TXN once (`PXT_OP_QUERY_ID` on slot 0 — the TXN cap is
pre-opened by the spawning shell, mkdir never mints one) before walking
`pos_ptrs[0..pos_count]`, then calls `mkdir_one` per path,
**fail-fast**: the first non-zero return is recorded in `exit_code`
plus `err_pos_index` and the walk stops *without* committing, so
nothing any positional staged is persisted. `mkdir_one` no longer opens
or commits a TXN itself — it only walks path components. Per component
it probes for pre-existence (`PFF_OP_QUERY_INODE` on slot 1), creates
(`PXT_OP_CREATE` = 9 on slot 0), stamps the owner (`USER_OP_QUERY_FP0`
on slot 2), and stages one `CreatedDirRecord` + one `RmdirUndoRecord`
at an index in a running, invocation-wide `newly_created_count` (so
`mkdir a b c` stages three of each, not one overwritten three times).
Once every positional returns `MK_OK`, `mkdir_run` freezes both record
counts to `newly_created_count` *before* the single `PXT_OP_COMMIT`
(= 7) syscall, so a commit failure leaves records and undo log in a
consistent "staged, never committed" state. Pre-existing levels skip
create, do not advance `newly_created_count`, and get no records — so
undo of `mkdir -p a/b/c` where `a` existed removes only `a/b/c` and
`a/b`.

Without `-p`, any `/` in the path is rejected with
`MK_MULTI_LEVEL_UNSUPPORTED` (POSIX-canonical). With `-p`,
`mkdir_split_path` decomposes the path: absolute paths are refused
(`MK_ABS_PATH_UNSUPPORTED` — v1.0 writes only inside the invoker's own
subtree, no libpdx-elevate hop), empty components are skipped
(`foo//bar` → 2 levels, `foo/` → 1), `.` components are *not*
normalised, and more than `PATH_MAX_COMPONENTS` = 16 non-empty levels
returns `MK_PATH_TOO_DEEP`. An empty path yields `comp_count == 0` and
short-circuits to `MK_OK` (POSIX-idempotent `mkdir -p ""`).

**v1.0 substrate status.** The create / probe / stamp / commit steps
dispatch through real cap kinds and real op ordinals, but the
kernel-side write bodies are the open `mkdir.M2-substrate-{001,002,003}`
seams at paideia-os — `PXT_OP_CREATE` currently returns `PXT_STUB_OK`.
Likewise, `_start` builds a **static bootstrap argv**
(`-p -v --dry-run a/b/c`) rather than reading argv from the shell; the
real argv-in-sidecar wire waits on shell.M4. Signatures are frozen, so
the substrate lands as body edits only.

## Options

Only three flag names are byte-matched in `parse_flags_from_argv`
(`src/mkdir.pdx`). Anything else is parsed by libpdx-argv and then
**silently ignored** — the standard-vocabulary rejection path lands at
libpdx-argv.M2-002. In particular, the `--json`, `--schema`, `--help`
and `--version` flags described in `doc/mkdir.pdxdoc` are not
recognised by the v1.0 recogniser.

| Flag | State slot | Effect |
|---|---|---|
| `-p` | `flag_p` | Create parent directories. Lifts the multi-level `/` guard and routes the path through `mkdir_split_path`; every level lives in one TXN scope; pre-existing levels are no-ops, not errors. |
| `-v` | `flag_v` | Parsed and stored, but **emits nothing at v1.0** — the verbose emission in `mkdir_one` Step 3 is still deferred. |
| `--dry-run` | `flag_dry_run` | Short-circuit to `MK_OK` *after* the path split has validated the path, before the TXN open. `mkdir -p --dry-run /abs` still returns `MK_ABS_PATH_UNSUPPORTED`. |

## Exit codes

From `MkdirState::MK_*` in `src/mkdir_state.pdx`; `_start` passes
`exit_code` straight to `sys_exit`.

| Code | Constant | Meaning |
|---|---|---|
| 0 | `MK_OK` | Success (or `--dry-run`, or empty path). |
| 1 | `MK_ARGV_ERR` | `parse_argv` returned non-zero. |
| 2 | `MK_MISSING_PATH` | `pos_count == 0`. |
| 3 | `MK_MULTI_LEVEL_UNSUPPORTED` | Path contains `/` and `-p` was not passed. |
| 4 | `MK_TXN_OPEN_FAIL` | `sys_cap_invoke(SLOT_TXN, PXT_OP_QUERY_ID)` returned negative. |
| 5 | `MK_MKDIR_FAIL` | Pre-existence probe or `PXT_OP_CREATE` returned negative. |
| 6 | `MK_TXN_COMMIT_FAIL` | `PXT_OP_COMMIT` returned negative (e.g. `PXT_BAD_TRANSITION`). |
| 7 | `MK_ABS_PATH_UNSUPPORTED` | Path starts with `/`. |
| 8 | `MK_PATH_TOO_DEEP` | More than 16 non-empty components. |
| 9 | `MK_CAP_TAIL_FAIL` | `KIND_USER` owner stamp returned negative. |

Every failure path also writes a one-line diagnostic to stderr through
`emit_stderr` (`sys_debug_puts`, SC+ ID 12) before returning.

## Capabilities

Effect and capability rows, verbatim from the function declarations:

    emit_stderr           : (u64, u64) -> u64  !{mem, sysreg} @{}
    emit_audit_event      : (u64) -> u64       !{mem, sysreg} @{cap}
    parse_flags_from_argv : () -> ()           !{mem}         @{}
    mkdir_split_path      : (u64) -> u64       !{mem}         @{}
    mkdir_one             : (u64) -> u64       !{mem, sysreg} @{cap}
    mkdir_run             : () -> ()           !{mem, sysreg} @{cap}
    _start                : () -> ()           !{mem, sysreg} @{sched}
    mkdir_state_reset     : () -> ()           !{mem}         @{}

Cap sidecar — `caps.decl` declares four required rows; `_init_caps`
seeds five (`_init_caps_count = 5`), the fifth added at M3-002:

| Slot | Kind | Rights | Use |
|---|---|---|---|
| 0 | `KIND_PDXFS_TXN` | `INVOKE \| OBSERVE` (0x408) | Pre-opened TXN; query-id, create, commit. |
| 1 | `KIND_PDXFS_FILE` | `INVOKE` (0x008) | Target parent inode; pre-existence probe. |
| 2 | `KIND_USER` | `INVOKE` (0x008) | Invoker cap stamped as owner tail. |
| 3 | `KIND_IPC_ENDPOINT` | `WRITE \| INVOKE` (0x00A) | Semantic pipe (`CreatedDirRecord@0.1`). |
| 4 | `KIND_IPC_ENDPOINT` | `WRITE \| INVOKE` (0x00A) | `svc.audit-journal`. Kept separate from slot 3 so revoking the schema pipe cannot revoke audit. |

`target_ptr` is 0 in every sidecar row — the spawning shell fills it via
`sys_cap_transfer` before the exec landing.

## Examples

    # Single level. No '/', so the -p guard passes; one component created.
    $ mkdir a
    $ echo $?
    0

    # Multi-level without -p — rejected before any cap invocation.
    $ mkdir src/kernel
    mkdir: multi-level path needs M2 (-p)
    $ echo $?                                     # MK_MULTI_LEVEL_UNSUPPORTED
    3

    # Multi-level atomic: three levels in one TXN, three records staged.
    $ mkdir -p a/b/c
    $ echo $?
    0

    # Dry run still validates: the split rejects the absolute path
    # before the short-circuit, so this is not exit 0.
    $ mkdir -p --dry-run /system/audit
    mkdir: absolute path unsupported at M2
    $ echo $?                                     # MK_ABS_PATH_UNSUPPORTED
    7

    # Multi-positional: one shared TXN across every operand
    # (mkdir.ENH-006). Three CreatedDirRecord + three RmdirUndoRecord
    # entries (indices 0..2, one per operand), one PXT_OP_COMMIT.
    $ mkdir a b c
    $ echo $?
    0

## Audit records

mkdir emits to the audit journal and stages two record families.

**Audit-journal events** (`emit_audit_event`, slot 4). One
`sys_cap_invoke` with `op_arg` packing `IPC_OP_SEND` (3) in bits [7:0]
and the event kind in bits [63:8]. The return is **always ignored** —
per D3 a saturated journal must not sink the mkdir op.

| Kind | Value | When |
|---|---|---|
| `UEJ_KIND_TOOL_INVOKE` | 130 | Once at `_start`, before any argv work; sets `audit_emitted_invoke` so re-entry cannot double-emit. |
| `UEJ_KIND_TOOL_ERROR` | 131 | Before each of the three non-zero exits: argv parse failure, missing operand, and any non-zero `exit_code` from `mkdir_run`. |

**`CreatedDirRecord@0.1`** — handle `0x0100000100010001`, 4 × u64, one
per newly-created level, up to `created_dir_records_count`:

| Offset | Field | Value |
|---|---|---|
| +0 | `path_ptr` | Base of the full path string in argv. |
| +8 | `path_len` | `comp_start_offsets[i] + comp_lengths[i]` — the prefix naming *this* level. |
| +16 | `parent_txn_id` | `current_txn_id`; shared by every record in one invocation. |
| +24 | `owner_slot_ref` | 2 (`SLOT_USER`); resolved to `user_fp_lo` by libpdx-cap's `KIND_USER_ref` decode at render time. |

**`RmdirUndoRecord@0.1`** — handle `0x0100000600010001`, 3 × u64
(`path_ptr`, `path_len`, `parent_txn_id`), one per newly-created level,
up to `rmdir_undo_records_count`. Replay is `rmdir <path>` in **reverse**
insertion order (inner → outer), so each level is empty when removed.
No owner field: rmdir does not consult owner at replay. Both arrays are
bounded at 16 records and are consumed by index, so trailing entries are
never zeroed.

Marshalling into a durable `/system/audit/user-events/*.pdxevent` record
waits on the `svc.audit-journal` daemon body; at v1.0 the send lands in
the R12 IPC MVP global channel, which still exercises the dispatch and
the `WRITE | INVOKE` rights gate.

## See also

- [`libpdx-argv`](https://github.com/paideia-os/libpdx-argv) — `parse_argv`, `reset`, `ParsedArgs`
- [`cp`](https://github.com/paideia-os/cp) · [`mv`](https://github.com/paideia-os/mv) · [`rm`](https://github.com/paideia-os/rm) — sibling R50 coreutils
- `doc/mkdir.pdxdoc` (`doc mkdir`) — full man page, including the flags planned beyond v1.0
- `design/architecture.md` — module boundary, `_start` shape, `mkdir_one` sequence, sidecar slot map
- `CHANGELOG.md` / `STATUS.md` — release history and per-issue milestone table
- `design/tooling/r49-r50-plan.md` §5.9 in [paideia-os](https://github.com/paideia-os/paideia-os) — wave plan

## License

MIT — see LICENSE.
