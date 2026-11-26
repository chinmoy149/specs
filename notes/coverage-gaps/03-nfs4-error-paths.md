<!--
SPDX-FileCopyrightText: 2026 The Quint Specs Authors
SPDX-License-Identifier: MIT
-->
# 3. Stale and bad filehandles, and other thin error paths

**Model:** nfs4 · **Size:** small each · **Status:** todo

## What it is

Many nfs4 operations are well covered on success and thinly covered on
failure. The clearest case is the filehandle. The model never sends an
invalid one, so the errors a server must give for a stale or garbage handle
are never tested.

## What the model has today

- **PUTFH always succeeds.** `opPutfh` (`quint/nfs/nfs4_ops.qnt:1097`) sets
  the current filehandle and returns `OK`. Its comment says: "The generator
  only presents live inos; a removed ino would surface STALE here or at first
  use (R-CORE-102 leaves the position open)."
- **STALE exists but is unreachable in practice.** The shared check `fhGate`
  (`nfs4_ops.qnt:676`) returns `E_STALE` for a removed inode. But since the
  generator only hands out live inodes, the only way to reach it is a handle
  that became stale inside one compound.
- **BADHANDLE is defined and never used.** `E_BADHANDLE = 10001`
  (`nfs4_ops.qnt:59`) has no other reference.
- **Other thin spots.** In `quint/nfs/nfs4_coverage_baseline.txt`, 21
  operations have one bucket, including GETATTR, ACCESS, CLOSE, LOCKU, PUTFH,
  SAVEFH and VERIFY. Those baselines are minimums, though (see the caveat in
  the [tracker](README.md)). Measure before assuming a path is unreached.

## What we'll add

Do this in two steps.

**Step 1: measure.** Generate the nfs4 corpus (`cmake --build build`, or
`tools/gen.js` for the nfs4 family) and run
`quint/nfs/coverage.py --glob '...' --low 2`. List every operation whose only
observed bucket is `OK`. That list replaces the guesswork above.

**Step 2: fill the filehandle gap first.**

1. **Model.** Let the generator present a filehandle for an inode it removed
   earlier in the run. Then a later PUTFH, or the op after it, must return
   `STALE`. RFC 7530 §16.20 / RFC 8881 §18.19 let the server report it at
   PUTFH or at first use, so the reply needs an accept set. `SRename` already
   does this with `stAccept` (`nfs4_ops.qnt:1516`).
2. **Model.** Add a garbage-handle case that answers `BADHANDLE`.
3. **Generator.** A `putfhStale` and a `putfhBad` action.
4. **Self-tests.** One per outcome.
5. **Harness.** The replayer must remember the wire handle of a removed
   object, and be able to send bytes no server would issue.

Then go through the step 1 list one op at a time.

## Open questions

- Can a harness keep a handle across a REMOVE? It must not reuse the inode.
  On some servers a removed inode number comes back quickly.
- Is `STALE` at PUTFH or at first use a deviation, or should the model accept
  either? Accepting either is safer, since the RFC leaves it open.

## Done when

- Step 1's list exists and is in this doc.
- PUTFH with a removed handle, and with a garbage handle, are generated,
  tested and replayed.
