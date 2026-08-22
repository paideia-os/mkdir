# .pkgs-mirror/ — mirror push staging

Meta-files documenting the M5-002 push contract for
`https://pkgs.paideia-os/` (T-INFRA-001 in
`design/tooling/plan.md` §10, paideia-os). Nothing here ships in
`pkg.tar` (see `staging.pdxpush` [tar-layout] block).

## Files

- `staging.pdxpush` — mirror-push manifest. Four sections:
  `[tar-layout]` (files that go into pkg.tar), `[pkgs-tree]` (final
  mirror-host layout after Phase 2 promote), `[push-metadata]`
  (placeholder rows filled by the signing bot at push time),
  `[smoke-runbook]` (post-push steps run by the reviewer on a fresh
  paideia-os VM).

## Two-phase push flow

    Phase 1 — Author push to staging/
      $ paideia-as release --sign      # produces pkg.tar + author-signed manifest.pdxsig
      $ paideia-as push staging         # HTTPS PUT to pkgs.paideia-os/staging/mkdir/1.0.0/

    Phase 2 — Bot re-sign to main/ (human-in-the-loop)
      Bot polls staging/, opens review UI, on approve:
        - Re-signs manifest.pdxsig under paideia_root_pk.
        - Copies pkg.tar + manifest.pdxsig to main/mkdir/1.0.0/.
        - Updates + re-signs main/index.pdxsig.

## v1.0 status

At v1.0 the mirror host is not yet stood up. `staging.pdxpush` is the
release-script contract that goes live the moment `pkgs.paideia-os`
serves. Placeholder-substrate discipline mirrors the M4 QEMU smoke
fixtures — the shape freezes here, the wire lands later without a
signature change.
