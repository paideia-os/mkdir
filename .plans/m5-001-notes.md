# mkdir.M5-001 — implementation notes

**Issue:** #14 — dual-signed release + .pdxdoc.
**Upstream doc:** `design/tooling/r49-r50-plan.md` §5.9 (paideia-os);
`design/tooling/plan.md` §D4 + §6.4 + §9.3 (release layout + signing
pipeline).

## What landed

- `doc/mkdir.pdxdoc` — full man-equivalent per plan.md §I7. Sections:
  NAME, SYNOPSIS, DESCRIPTION, FLAGS, EXIT CODES, CAPABILITIES,
  OUTPUT SCHEMA, DIFFERENCES FROM POSIX, EXAMPLES, FILES, SEE ALSO,
  HISTORY. Rendered by the `doc` tool from a running shell as
  `doc mkdir`; also served as the `--help` back-end per I3 via the
  shell's help-dispatch path.
- `manifest.pdxsig` — dual-signed release manifest per §D4 with three
  sections: `[manifest]` (per-file SHA-256 hash table + [capabilities]
  + [schemas] + [deps] sub-blocks), `[author-sig]` (ML-DSA-65 sig
  under author_pk), `[paideia-sig]` (ML-DSA-65 sig under
  paideia_root_pk). Both signature blocks carry placeholder bytes at
  v1.0 — the paideia-signing-bot host (T-INFRA-002) replaces them at
  M5-002 mirror-push time without editing the manifest body.
- `deps.list` — semver-pinned shared-library dependencies. Rows for
  libpdx-argv, libpdx-cap, libpdx-audit, libpdx-semantic-pipe all at
  1.0.0. Fingerprint columns carry placeholders until the signing bot
  is up. libpdx-elevate intentionally NOT listed — mkdir writes only
  inside the invoker's subtree at v1.0.
- `CHANGELOG.md` — 1.0.0 entry per plan.md M5 rubric line
  "CHANGELOG-1.0 entry". Documents the four milestones landed
  (M1..M4) plus the M5-001 release scaffolding, cross-repo
  dependency version pin, placeholder-substrate seams still open,
  known v1.0 limitations, and the signing-pipeline status.

## Placeholder-until-signing-bot rationale

The paideia-signing-bot host (T-INFRA-002 in `design/tooling/plan.md`
§10) is not yet stood up: neither `author_pk` for `paideia-os-team`
nor `paideia_root_pk` (R32-era) has been generated as HEAD of the
paideia-os repo, and `pkgs.paideia-os` (T-INFRA-001) is not yet
serving. That means real ML-DSA-65 signatures cannot land at this
commit.

The frozen-shape strategy is the same one M4 uses for the
placeholder-substrate seams: land the file layout, hash contents that
DO exist (source, tests, caps.decl, deps.list, doc, LICENSE, README,
design/architecture.md) into the `[manifest]` block by their real
SHA-256, and stub the two `[*-sig]` blocks with PLACEHOLDER text
whose line count matches a real ML-DSA-65 signature (5 lines of 64
base64 chars each covers the 3293-byte signature envelope).

At the M5-002 mirror-push step, the signing bot will:

1. Re-hash the same 17 files listed in `[manifest]` and confirm the
   hashes match this file byte-for-byte.
2. Canonically encode the byte range from after `[manifest]` to before
   `[author-sig]`.
3. Sign that range twice — once with `author_pk`, once with
   `paideia_root_pk`.
4. Overwrite the two `[*-sig]` blocks with the real signatures. The
   `[manifest]` block MUST NOT be touched at this step or the hash
   chain breaks.
5. Push to `pkgs.paideia-os/staging/mkdir/1.0.0/` per §6.3, then
   promote to `main/` after human review.

## Hash-table invariant

The `[manifest]` block lists every file that ships in `pkg.tar` and
NOTHING else. Verification order:

```
for row in [manifest]:
    computed = sha256(read_file(row.path))
    if computed != row.hash:
        reject(EXIT_5_SIGNATURE)
verify_ml_dsa_65([author-sig], canonical([manifest]), author_pk)
verify_ml_dsa_65([paideia-sig], canonical([manifest]), paideia_root_pk)
```

This is cheaper than a Merkle tree walk at install time (each row is
one file open + one hash) and gives the same tamper-evidence: any
byte change in any file forces a `[manifest]` rewrite which forces a
signature rewrite. `pkg verify --caps-only` short-circuits by reading
only the `[capabilities]` sub-block — used by `caps --audit <pkg>` at
runtime without touching the source files.

## Files that ship in pkg.tar

Per the `[manifest]` block (17 rows):

- `src/mkdir.pdx`, `src/mkdir_state.pdx`
- `tests/test_m4_00[1-4]_*.pdx` (4 files)
- `tests/expected-mkdir-m4-00[1-4]-*.txt` (5 files including the two
  M4-001 witnesses — single + multi)
- `caps.decl`, `deps.list`, `doc/mkdir.pdxdoc`
- `LICENSE`, `README.md`, `design/architecture.md`

Files that do NOT ship in pkg.tar:

- `.plans/*` — implementation notes are per-repo, not per-package.
- `STATUS.md` — repo-scope status, mirrors CHANGELOG.md.
- `.git/*` — obviously.
- `manifest.pdxsig` itself — the signature sits BESIDE pkg.tar in
  the pkgs.paideia-os layout, not inside it (per §6.3).

## paideia-as conformance

None of the M5-001 files are `.pdx` source — no paideia-as encoding
concerns. The `manifest.pdxsig` file is plain ASCII with a leading
`%pdxsig 0.1` header that the future `pkg verify` binary treats as
the format-version tag. Same shape as the `%pdxdoc 0.1` header in
`doc/mkdir.pdxdoc`.

## What did NOT land at M5-001

- **Real ML-DSA-65 signatures.** Deferred to M5-002 mirror-push step,
  which itself is gated on T-INFRA-001 (mirror host) + T-INFRA-002
  (signing bot). See CHANGELOG.md "Signing pipeline status".
- **`pkg.tar` archive itself.** The manifest lists what would be in
  it; the archive is produced by `paideia-as release --sign` per §9.3
  step 2. That tool (paideia-as `release` subcommand) is a paideia-as
  bug tracked separately from this repo.
- **`--version` real output.** The tool binary at HEAD would print
  the placeholder fingerprint from `manifest.pdxsig`; when the signing
  bot lands, the fingerprint gets substituted and the `--version` line
  reads correctly without a source change.
- **`--tutorial` and `--examples`.** Per plan.md §I7, every shipping
  tool needs all four of `--help`, `doc <tool>`, `<tool> --tutorial`,
  `<tool> --examples`. mkdir's `--help` and `doc mkdir` land at M5-001;
  the tutorial + examples gallery is out of scope for v1.0 and files
  as post-1.0 follow-ups (roadmap issue in paideia-os, not this repo).

## Build-note

Not run per repo memory rule `feedback_no_background_builds`. M5-001
touches no `.pdx` source — no build regression risk. The `[manifest]`
hash table was computed via `sha256sum` at authoring time; the two
new files (`doc/mkdir.pdxdoc`, `deps.list`) have their hashes as
computed. If any pre-M5-001 file is edited before M5-002 push, the
`[manifest]` block MUST be re-computed before re-signing.
