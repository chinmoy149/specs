<!--
SPDX-FileCopyrightText: 2026 The Quint Specs Authors
SPDX-License-Identifier: MIT
-->
# 10. NFSv4 server restart and state reclaim

**Model:** nfs4 · **Size:** large · **Status:** todo

## What it is

When an NFSv4 server restarts, it loses its in-memory state: clients, opens,
locks, delegations. It then enters a **grace period**. During it, clients
reclaim what they held, using:

- OPEN with `CLAIM_PREVIOUS`
- LOCK with `reclaim = true`
- RECLAIM_COMPLETE in 4.1, to say they're done

Meanwhile the server refuses new conflicting state with `NFS4ERR_GRACE`.
After grace, reclaims get `NFS4ERR_NO_GRACE`. References: RFC 8881 §8.4.2
(server failure and recovery) and RFC 7530 §9.6.2.

This is where servers have real, data-losing bugs. For example, a lock
granted to a new client during grace that conflicts with a lock another
client is about to reclaim.

## What the model has today

- **No restart event.** Lease loss is modelled as an explicit event, but the
  server never loses state.
- **RECLAIM_COMPLETE is modelled** (`opReclaimComplete`,
  `nfs4_ops.qnt:1976`), but without grace there's nothing for it to end.
- **The authors have deferred this on purpose.** `opFreeStateid`'s comment
  (`nfs4_ops.qnt:1991`) says: "Freeing revoked delegations/layouts is part of
  the deferred restart scenarios."
- `quint/README.md` lists "state reclaim after a server restart" as left
  out.

## What we'll add

This is a design task before it's a coding task. Talk to the maintainers
before writing much model code.

1. **A restart event.** It moves the current state into a "previous" record
   (what each client may reclaim) and empties the live state. A grace flag
   turns on.
2. **Reclaim operations.** OPEN `CLAIM_PREVIOUS` and LOCK `reclaim` succeed
   only for state in the previous record. Otherwise they fail with
   `RECLAIM_BAD`, or `RECLAIM_CONFLICT` if another client got there first.
3. **Grace rules.**
   - Non-reclaim OPEN and LOCK get `GRACE`.
   - READ, WRITE and SETATTR on conflicting state get `GRACE`.
   - After grace ends, reclaims get `NO_GRACE`.
4. **End of grace.** It ends when every client has sent RECLAIM_COMPLETE, or
   on a timeout. The timeout is abstract, like all time in the model.
5. **Invariant.** No state granted during grace conflicts with any reclaimable
   state. That's the safety property that catches real bugs.
6. **Harness.** The hard part. The replayer has to restart the server under
   test mid-trace, which each harness does differently (knfsd, ganesha,
   chimera). It may need a harness hook that some servers report as SKIP.

## Open questions

- Is restart a feature constant (`F_RESTART`), since not every harness can
  restart its server?
- 4.0 and 4.1 reclaim differ: 4.0 has no RECLAIM_COMPLETE. Do both, or 4.1
  first?
- How does it interact with the duplicate request cache? The nfsdrc model
  also says nothing about restart.

## Done when

- A design note agreed with the maintainers, then the restart event, grace
  rules and invariant modelled and tested. Harness support may follow in a
  separate PR.
