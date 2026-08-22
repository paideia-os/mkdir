# tests/

Empty at M1 by design. The coreutil test matrix — single-level,
multi-level `-p`, mixed pre-existing + new under `-p`, TXN-abort
mid-create, cap-tail correctness — lands with `mkdir.M4-001` through
`mkdir.M4-004` per `design/tooling/r49-r50-plan.md` §5.9 in paideia-os.

The M1 first-runnable smoke witness is the `mkdir: M1 ring3 ok\n` line
that `_start` emits before its `sys_exit` — same proof-of-life shape as
`R31 ECHO CLIENT RING3 OK` in `src/user/echo_client.pdx` (paideia-os).
Its acceptance ships as part of the paideia-os smoke matrix, not from
this repo's `tests/` tree.
