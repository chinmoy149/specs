<!--
SPDX-FileCopyrightText: 2026 The Quint Specs Authors
SPDX-License-Identifier: MIT
-->
# 5. renameat2 flags and flock(2) in the posix model

**Model:** posix · **Size:** medium · **Status:** todo

These are two separate pieces of work, kept in one doc because both are
Linux-specific calls missing from the same model. They can land in separate
PRs.

## 5a. renameat2 flags

### What it is

`renameat2(2)` is `rename` with flags:

- `RENAME_NOREPLACE`: fail with `EEXIST` instead of replacing the target.
- `RENAME_EXCHANGE`: swap the two names atomically. Both must exist.
- `RENAME_WHITEOUT`: overlayfs-only. Skip it.

### What the model has today

- `RRename({ dfdOld, pthOld, dfdNew, pthNew })`
  (`quint/posix/posix_ops.qnt:539`), with no flags field.
- "NOREPLACE" and "EXCHANGE" appear nowhere in `quint/posix/`.

### What we'll add

1. **Model.** A `flags` field on `RRename`, or a separate `RRename2`.
   NOREPLACE is a small change: one more check before the existing replace
   logic. EXCHANGE is the interesting part: two directory entries swap, so a
   file and a directory can trade places, and directory link counts change.
2. **Errors.** `EEXIST` (NOREPLACE), `ENOENT` (EXCHANGE with a missing
   target), `EINVAL` (both flags at once), and `EINVAL` when the filesystem
   doesn't support the flag. That last one is probably a feature constant.
3. **Self-tests, generator and harness.** Python has no `renameat2` wrapper,
   so `harness/posix/posix_driver.py` needs `ctypes` or a `syscall` call.

## 5b. flock(2)

### What it is

BSD-style whole-file locks: `LOCK_SH`, `LOCK_EX`, `LOCK_UN`, optionally with
`LOCK_NB`. On Linux, `flock` locks belong to the open file description and
are independent of `fcntl` record locks.

### What the model has today

- `fcntl` locks (`RFcntlLock`) and `lockf` (`RLockf`) are modelled
  (`posix_ops.qnt:550`), including OFD locks.
- `flock` is not a request kind.

### What we'll add

1. **Model.** `RFlock({ fd, op, nb })`. The lock lives on the open file
   description, so `dup` shares it and a separate `open` doesn't. The model
   already tracks descriptions for OFD locks, so reuse that.
2. **Behaviour to get right.**
   - `LOCK_NB` gives `EWOULDBLOCK` on conflict.
   - Converting shared to exclusive is not atomic.
   - Closing the last descriptor of the description releases the lock.
   - `flock` and `fcntl` locks don't conflict on Linux local filesystems.
     They do on NFS, which is a nice cross-model note.
3. **Self-tests, generator and harness.** `fcntl.flock` in Python.

## Open questions

- Without non-blocking mode, a blocked `flock` needs the model's blocking
  machinery. How does `CSetlkw` handle it today? Copy that.
- Is EXCHANGE across directories worth it in the first PR, or same-directory
  only?

## Done when

- Tests for every outcome above, the posix `_run` instances simulate clean,
  and the ext4 harness replays both.
