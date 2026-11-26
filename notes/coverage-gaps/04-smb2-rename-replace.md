<!--
SPDX-FileCopyrightText: 2026 The Quint Specs Authors
SPDX-License-Identifier: MIT
-->
# 4. SMB2 rename that replaces an existing name

**Model:** smb2 · **Size:** medium · **Status:** todo

## What it is

An SMB2 rename is a SET_INFO with `FileRenameInformation`. It carries a flag,
`ReplaceIfExists`. When the flag is false, a rename onto an existing name
fails with `STATUS_OBJECT_NAME_COLLISION`. When it is true, the existing file
is replaced, unless something stops it. MS-FSA §2.1.5.14.11 defines both.

## What the model has today

- Rename is `CSetRename({ fid, newdir, newname })`
  (`quint/smb2/smb2_ops.qnt:283`). There's no replace flag.
- The evaluator models only the no-replace case. Its comment
  (`smb2_ops.qnt:2209`) gives the reason: "Modeling the no-replace case keeps
  the target's identity/handles intact -- replacing a still-open directory
  target is exactly what the fs core cannot represent, and MS rename does not
  replace directories anyway."
- The access check runs before the collision check, and recorded deviation
  SD-4 (Samba reports the collision first) depends on that order. Don't break
  it.

## What we'll add

1. **Model.** Add `replace: bool` to `CSetRename`. With `replace = true`:
   - The target is a file with no open handles: it is removed and the rename
     succeeds.
   - The target is a file with open handles: the rename fails. Check
     MS-FSA §2.1.5.14.11 for the exact status (probably
     `STATUS_ACCESS_DENIED`).
   - The target is a directory: the rename fails.
   - The source and target are the same file (a hard link, or the same name):
     check the spec.
2. **Model.** The replaced file's other names (hard links) survive, so only
   the target directory entry goes.
3. **Leases and oplocks.** Replacing a file that someone holds a handle lease
   on should break the lease first. Check what `smb2_state.qnt` does for
   delete-on-close, since that's the closest existing case.
4. **Notify.** A replace fires notifications in `smb2_notify.qnt`: the old
   target removed and the rename. Check which ones the model already emits.
5. **Self-tests and harness.** Tests in `smb2Test.qnt`. In
   `harness/samba/smb2_wire.py`, set the flag in the rename request.

## Open questions

- The fs core shared with nfs4 (`quint/nfs/nfs4_fs.qnt`) already does a
  replacing rename for NFS. Can smb2 reuse it, limited to file targets with no
  open handles?
- Does Samba follow MS-FSA here? If not, that's a new deviation in
  `quint/smb2/corpus.schema.json`.

## Done when

- Every replace outcome above has a test, and the `_run` instances simulate
  clean.
- The Samba suite (`ctest -L samba`) passes, or any new divergence is
  recorded with a citation.
