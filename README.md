# mkdir

paideia-os directory create — a R50 coreutil.

## Install

    $ pkg install mkdir

Verifies the dual-signed `manifest.pdxsig` (ML-DSA-65 under
`author_pk` + `paideia_root_pk`) before extracting into
`/pkgs/mkdir-1.0.0/` and symlinking `/bin/mkdir`.

## Usage

    $ mkdir [-p] [-v] [--dry-run] [--json] [--schema] <path> [<path>...]

See `doc mkdir` (or `doc/mkdir.pdxdoc` in-repo) for the full man page
including EXIT CODES, CAPABILITIES, OUTPUT SCHEMA, DIFFERENCES FROM
POSIX, and EXAMPLES.

## Status

**v1.0.0** — first signed release (M5-001 LANDED, M5-002 mirror push
queued). See `CHANGELOG.md` for the milestone rollup and `STATUS.md`
for the per-issue table.

Design source: `design/tooling/r49-r50-plan.md` §5.9 in
[paideia-os](https://github.com/paideia-os/paideia-os).

## License

MIT — see LICENSE.
