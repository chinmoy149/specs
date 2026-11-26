<!--
SPDX-FileCopyrightText: 2026 The Quint Specs Authors
SPDX-License-Identifier: MIT
-->
# 2. Extended attributes in the posix model

**Model:** posix · **Size:** small-medium · **Status:** todo

## What it is

Linux extended attributes: `setxattr`, `getxattr`, `listxattr` and
`removexattr`, plus their `l*` (don't follow symlinks) and `f*`
(file-descriptor) forms. They are named values attached to a file. They are
Linux behaviour, not POSIX, which fits this model: it already covers
"POSIX.1-2024 + Linux". Reference: `xattr(7)` and the man pages for each
call.

## What the model has today

- None of the four calls is in `OpReq` (`quint/posix/posix_ops.qnt:493`),
  which lists about 60 other calls.
- The inode record in `quint/posix/posix_fs.qnt` (line 97) has no xattr
  field.
- There's a pattern to copy. The nfs4 model has xattrs on its inodes
  (`xattrs: Name -> int` in `quint/nfs/nfs4_fs.qnt:83`) and evaluators
  `opGetxattr`, `opSetxattr`, `opListxattrs` and `opRemovexattr`
  (`quint/nfs/nfs4_ops.qnt:3653`).

## What we'll add

1. **Model.** An `xattrs: Name -> int` field on the posix inode. Values stay
   abstract, the same as file contents. Add request kinds
   `RSetxattr({ dfd, pth, follow, name, sym, flags })`, `RGetxattr`,
   `RListxattr`, `RRemovexattr`, and fd forms if worth it. Errors to model:
   `ENODATA` (missing name), `EEXIST` / `ENODATA` for `XATTR_CREATE` /
   `XATTR_REPLACE`, `ERANGE` (buffer too small), `EACCES`, `ENOTSUP`, and the
   usual path-resolution errors.
2. **Generator.** Actions for each call with a small name universe, two or
   three names, so collisions happen.
3. **Self-tests.** One per outcome in `quint/posix/posixTest.qnt`.
4. **Feature constant.** Some filesystems don't support user xattrs, so this
   probably belongs behind an `F_XATTR` feature in
   `quint/posix/corpus.schema.json`, as nfs4 does it.
5. **Harness.** `os.setxattr` and friends in `harness/posix/posix_driver.py`,
   a replay arm in `posix_replay.py`, and config updates under
   `harness/posix/configs/`.

## Open questions

- Which namespace to test: `user.` only, or also `trusted.` and `security.`,
  which need privilege? Start with `user.`.
- Do symlinks matter? `lsetxattr` on a symlink in `user.` gives `EPERM` on
  Linux. That's worth one case.
- Does `rename` or `link` need changes? xattrs belong to the inode, so a
  hard link should see the same set. That makes a nice invariant.

## Done when

- `posixTest` passes with the new tests.
- The posix `_run` instances still simulate clean.
- The ext4 harness replays xattr traces.
