<!--
SPDX-FileCopyrightText: 2026 The Quint Specs Authors
SPDX-License-Identifier: MIT
-->
# 7. SMB2 QUERY_DIRECTORY

**Model:** smb2 · **Size:** large · **Status:** todo

## What it is

QUERY_DIRECTORY (MS-SMB2 §2.2.33, §3.3.5.18; behaviour in MS-FSA §2.1.5.5)
lists a directory through an open directory handle. Every Explorer window and
every `dir` uses it. The request carries:

- an information class: what each entry contains
- a search pattern with wildcards (`*`, `?`, and the DOS wildcards `<`, `>`,
  `"`)
- flags: `RESTART_SCANS`, `RETURN_SINGLE_ENTRY`, `INDEX_SPECIFIED`, `REOPEN`
- an output buffer length

The server keeps a cursor on the handle, so listing a big directory takes
several requests. It ends with `STATUS_NO_MORE_FILES`. A first call that
matches nothing gets `STATUS_NO_SUCH_FILE`.

## What the model has today

- Nothing. No QUERY_DIRECTORY command in `Cmd` (`quint/smb2/smb2_ops.qnt:254`)
  and no mention anywhere in `quint/smb2/`.
- What it can build on:
  - Directory opens exist (`isDir` in `CCreate`).
  - The fs core (`quint/nfs/nfs4_fs.qnt`) knows each directory's entries.
  - CHANGE_NOTIFY (`quint/smb2/smb2_notify.qnt`) already models an output
    buffer exactly, including the "one record too big" case. That's the
    pattern for buffer sizing here.

## What we'll add

Split it up, because one PR for all of this would be huge.

**7a: basic enumeration.**
1. **Model.** `CQueryDir({ fid, pattern, restart, single, obl })` with one
   information class, probably `FileNamesInformation` or
   `FileIdBothDirectoryInformation`. Per-handle cursor state in
   `smb2_state.qnt`. Enumerate `.` and `..` the way Windows does.
2. **Outcomes.**
   - Success with entries.
   - `NO_MORE_FILES` at the end.
   - `NO_SUCH_FILE` when nothing matches on the first call.
   - `INVALID_PARAMETER` on a non-directory handle.
   - `BUFFER_OVERFLOW` or `INFO_LENGTH_MISMATCH` for small buffers.
3. **Order.** The model can't promise an order, because servers don't. So
   check the set of names returned across a full enumeration, not the
   sequence. That needs a harness that collects across calls.

**7b: patterns.** `*`, `?` and literal names first; the DOS wildcards later.
Keep the name universe small, but include names that only some patterns
match.

**7c: concurrent changes.** Entries created or removed between calls. The
spec is loose here, so the model has to accept a range of answers.

## Open questions

- Which information class gives the most for the least? `FileNames` is
  simplest, but a Windows client mostly uses `FileIdBothDirectory`.
- How do traces express "this set of names over these calls"? This may need
  a new kind of check in the replayer.
- Is case-insensitive matching in scope? It depends on the share. Probably a
  policy constant.

## Done when

- 7a is modelled, tested and replays against Samba. 7b and 7c are tracked as
  follow-ups.
