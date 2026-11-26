<!--
SPDX-FileCopyrightText: 2026 The Quint Specs Authors
SPDX-License-Identifier: MIT
-->
# Quint coverage gaps

Places where the Quint models have no coverage, or very little, that are
worth adding. Each row links to a doc that says what the gap is, what the
model has today, and what adding it involves.

Gaps the models leave out on purpose are not listed: SMB2 authentication and
cryptography, file contents, real timestamps, and anything that needs a
crash. See [`quint/README.md`](../../quint/README.md) for those.

## Tracker

Status: `todo`, `scoping`, `in progress`, `in review` (with a PR link),
`done`.

| # | Gap | Model | Size | Status |
|---|-----|-------|------|--------|
| 1 | [NFSv4.1 TEST_STATEID](01-nfs4-test-stateid.md) | nfs4 | small | todo |
| 2 | [Extended attributes](02-posix-xattrs.md) | posix | small-medium | todo |
| 3 | [Stale and bad filehandles, and other thin error paths](03-nfs4-error-paths.md) | nfs4 | small each | todo |
| 4 | [Rename that replaces an existing name](04-smb2-rename-replace.md) | smb2 | medium | todo |
| 5 | [renameat2 flags and flock(2)](05-posix-renameat2-flock.md) | posix | medium | todo |
| 6 | [QUERY_INFO and SET_INFO classes](06-smb2-info-classes.md) | smb2 | medium | todo |
| 7 | [QUERY_DIRECTORY](07-smb2-query-directory.md) | smb2 | large | todo |
| 8 | [IOCTL / FSCTLs](08-smb2-ioctl.md) | smb2 | large | todo |
| 9 | [NFSv4 operations not in the model](09-nfs4-missing-ops.md) | nfs4 | varies | todo |
| 10 | [Server restart and state reclaim](10-nfs4-reclaim.md) | nfs4 | large | todo |

Rows are ordered roughly by how easy a first contribution is, not by value.
1 and 3 are good starting points; 7, 8 and 10 are the most valuable and the
most work.

## What "adding coverage" involves

Most gaps touch the same five places. A change that stops at the model is not
finished, because no trace will reach a server until the harness can send it.

1. **Model.** A request kind, a reply kind and a pure `op...` evaluator in the
   suite's `*_ops.qnt`, wired into the dispatcher.
2. **Generator.** An action in the suite's top-level `.qnt` that builds the
   request from live state, added to the step so simulation reaches it.
3. **Self-tests.** `run` tests in `*Test.qnt` for each outcome, success and
   every error the model can return.
4. **Coverage gate.** Where the suite has one (`coverage.py` plus a baseline
   file), raise the baseline with `--write-expect` once the new buckets appear.
5. **Harness.** Teach the replayer to encode the request and decode the reply
   (for example `harness/nfs/nfs4_client.py` and `nfs4_replay.py`).

If a server answers differently from the standard, record that as a deviation
in the suite's `corpus.schema.json`. Don't bend the model to match.

Read [`CONTRIBUTING.md`](../../CONTRIBUTING.md) before opening a PR.

## A caveat on the coverage baselines

A file such as `quint/nfs/nfs4_coverage_baseline.txt` is a minimum the corpus
must keep hitting. It does not list everything the corpus hits. FREE_STATEID
shows the difference: the baseline lists only `LOCKS_HELD`, yet the `lockFree`
generator (`quint/nfs/nfs4.qnt`) produces `OK` as well. To measure real
coverage, generate the corpus and run the suite's `coverage.py --low N`.

## Running things locally

```
quint test --main <TestModule> quint/<suite>/<suite>Test.qnt
quint run  --main <Instance> --max-samples 50 --max-steps 20 \
           --invariant inv quint/<suite>/<suite>_run.qnt
```

Most files hold several modules, so `--main` is required. List them with
`grep -n '^module' <file>`.