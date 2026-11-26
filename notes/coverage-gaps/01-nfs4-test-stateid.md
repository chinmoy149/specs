<!--
SPDX-FileCopyrightText: 2026 The Quint Specs Authors
SPDX-License-Identifier: MIT
-->
# 1. NFSv4.1 TEST_STATEID

**Model:** nfs4 · **Size:** small · **Status:** todo

## What it is

TEST_STATEID (RFC 8881 §18.48) takes a list of stateids and returns one
status for each, saying whether the server still considers it valid. A client
uses it after a lease problem to find out which of its state survived, before
deciding what to free or recover. It is the read-only partner of FREE_STATEID.

## What the model has today

- No `RTestStateid` request and no `STestStateid` reply in
  `quint/nfs/nfs4_ops.qnt`.
- Its neighbour is modelled: `RFreeStateid` (`nfs4_ops.qnt:342`),
  `opFreeStateid` (`nfs4_ops.qnt:1993`), and the generators `lockFree` and
  `freeStateidOpen` (`quint/nfs/nfs4.qnt`, around line 1612).
- The stateid machinery TEST_STATEID needs already exists: `SidKind` and the
  `freedSids` set in `quint/nfs/nfs4_state.qnt` (lines 140 and 154), and
  `sidLive`.
- A comment at `nfs4.qnt:1715` already mentions TEST_STATEID reporting a
  stateid valid, so the authors have it in mind.
- The harness has no opcode for it: `harness/nfs/nfs4_client.py` defines up to
  `OP_REMOVEXATTR = 75` but has no `OP_TEST_STATEID` (55).

## What we'll add

1. **Model.** `RTestStateid(List[SidRef])` and
   `STestStateid({ st: int, codes: List[int] })`. Return `OP_ILLEGAL` when
   `minor == 0`, as `opFreeStateid` does. For each stateid, return `OK` if
   live, `BAD_STATEID` if unknown or freed, `OLD_STATEID` if the seqid is
   behind, and `EXPIRED` or the `*_REVOKED` codes where the state model can
   tell. The op never changes state.
2. **Generator.** A `testStateid` action that picks a mix of live, freed and
   never-issued stateids, so every per-stateid code is reachable.
3. **Self-tests.** One `run` per code, plus one with a mixed list showing the
   codes come back in request order.
4. **Coverage.** Raise `nfs4_coverage_baseline.txt` once `STestStateid`
   buckets appear.
5. **Harness.** `OP_TEST_STATEID = 55`, an encoder and a decoder in
   `nfs4_client.py`; an `RTestStateid` arm in `nfs4_replay.py` next to the
   `RFreeStateid` one (line 623).

## Open questions

- The reply has a status per stateid. How should the bucket key look for the
  coverage gate: one bucket per distinct code seen, or per list shape?
- Which of the `*_REVOKED` codes can the state model distinguish today?
  `freedSids` only records "gone", not why.
- Does SEQUENCE need special handling? RFC 8881 says TEST_STATEID must not
  renew the lease or affect state.

## Done when

- `quint test` passes for `nfs4Test42` and `nfs4Test40` with the new tests.
- A short `quint run` on `nfs4Default` reaches TEST_STATEID with no invariant
  violation.
- The harness replays a TEST_STATEID trace against knfsd or ganesha.