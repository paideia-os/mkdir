# mkdir.M5-002 — implementation notes

**Issue:** #15 — mirror push (to `pkgs.paideia-os`).
**Upstream doc:** `design/tooling/r49-r50-plan.md` §5.9 (paideia-os);
`design/tooling/plan.md` §6.3 (repo model) + §9.3 (signing pipeline).

## What landed

- `.pkgs-mirror/staging.pdxpush` — mirror-push manifest with four
  sections:
    - `[tar-layout]` — the 17 files that go into `pkg.tar`
      (byte-frozen against `manifest.pdxsig` `[manifest]` block).
    - `[pkgs-tree]` — final mirror-host layout after Phase 2 promote.
    - `[push-metadata]` — placeholder rows filled at push time
      (release-commit hash, timestamp, pkg-tar sha256, author-fp,
      paideia-root-fp, reviewer handle, approve timestamp).
    - `[smoke-runbook]` — 8-step post-push runbook the reviewer runs
      on a fresh paideia-os VM.
- `.pkgs-mirror/README.md` — orientation doc for the mirror staging
  tree; explains the two-phase push flow and the v1.0 placeholder
  status.
- `STATUS.md` — M5-002 promoted to LANDED.
- `.plans/m5-002-notes.md` — this file.
- Git tag `v1.0.0` — attached to the M5-002 landing commit, matches
  the release-tag field in `.pkgs-mirror/staging.pdxpush` and the
  1.0.0 entry in `CHANGELOG.md`.

## Placeholder-until-mirror-stands-up rationale

Same rationale as M5-001's placeholder signatures: `pkgs.paideia-os`
is not yet a live HTTPS endpoint (T-INFRA-001 in
`design/tooling/plan.md` §10 remains open), so an actual `HTTPS PUT`
cannot happen at this commit. The shape-frozen strategy the M4 QEMU
smoke fixtures use applies again: land the push contract
byte-frozen, so when the mirror host comes up the release script
has nothing to invent.

The M5-002 landing IS the mirror-push contract — it declares which
files go into `pkg.tar`, what the mirror-tree URL layout is, which
metadata the signing bot fills, and what the reviewer smokes before
promote. When T-INFRA-001 stands up, a paideia-as release script
reads this file and runs the two-phase push against the live host.

## Why an 8-step smoke runbook not automated

The `[smoke-runbook]` block reads like a hand-run checklist, and
that's intentional. Per `design/tooling/plan.md` §9.3, the signing
bot is human-in-the-loop for first-party tools: "each first-party
tool release is manually re-signed after a reviewer confirms the
source-tree matches the author's tag." Automating the smoke runbook
before the reviewer role is fully documented would encode "no human
in the loop" into the pipeline — the opposite of the §9.3 policy.

Post-1.0, once several tools have gone through the pipeline and
common failure modes are catalogued, a subset of the runbook can
migrate into an automated pre-check the bot runs before showing the
reviewer the diff. That work is a paideia-signing-bot roadmap issue,
not a mkdir-repo issue.

## Coordination with M5-001

`.pkgs-mirror/staging.pdxpush` is READ-ONLY on the shipping files —
the `[tar-layout]` block lists the same 17 rows as
`manifest.pdxsig` `[manifest]` block, in the same alphabetical order.
A future refactor that adds or removes a file from `pkg.tar` MUST
update both files atomically or the pre-flight
`paideia-as release --verify-manifest` step fails at push time.

That constraint is enforced by a pre-push hook (paideia-os repo
convention per `feedback_paideia_os_no_cicd`): every push against
this repo checks that the union of `[manifest]` rows in
`manifest.pdxsig` equals the union of `[tar-layout]` rows in
`.pkgs-mirror/staging.pdxpush`. Diff → push refused with a
diagnostic naming the divergent file.

## Signing-pipeline dependencies (recap)

The M5-002 landing does not itself require T-INFRA-{001,002} to
close — the placeholder shape is authoritative until they do. The
graph is:

    M5-002 (this)          →  LANDED at v1.0.0 tag.
    T-INFRA-001 (mirror)   →  post-M5-002 (paideia-os wave R51+).
    T-INFRA-002 (bot)      →  post-T-INFRA-001.
    First real push        →  post-T-INFRA-002.

When the first real push happens, the placeholder rows in
`[push-metadata]` get their first non-PLACEHOLDER values, and
`manifest.pdxsig`'s `[author-sig]` + `[paideia-sig]` blocks get
their first real ML-DSA-65 signatures. Neither change requires a
mkdir code change.

## v1.0 tag

The tag `v1.0.0` is applied to the M5-002 landing commit rather than
the M5-001 commit because M5-002 is the LAST commit at v1.0 — the
mirror-push manifest is the final v1.0 artifact per the plan doc's
"released" criterion at end of M5. This matches the paideia-as
version-discipline pattern (`feedback_paideia_as_version_discipline`):
the tag moves at the phase close, together with the CHANGELOG entry
and the workspace version bump.

## What did NOT land at M5-002

- **Actual `HTTPS PUT` to pkgs.paideia-os.** Deferred — the host does
  not exist. See T-INFRA-001.
- **Actual signing-bot approval flow.** Deferred — the host does not
  exist. See T-INFRA-002.
- **The `paideia-as release --sign` and `paideia-as push` subcommand
  bodies.** These are paideia-as tool bugs, tracked separately in
  the paideia-as repo (not this repo).
- **`.pkgs-mirror/staging.pdxpush` schema versioning.** The
  `%pdxpush 0.1` header pins v0.1 of the format; schema evolution
  beyond that is a paideia-signing-bot roadmap concern.

## Build-note

Not run per repo memory rule `feedback_no_background_builds`. M5-002
touches no `.pdx` source and no file whose hash lives in
`manifest.pdxsig` `[manifest]` block — no manifest re-hash needed,
no regression risk.
