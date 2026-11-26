<!--
SPDX-FileCopyrightText: 2026 The Quint Specs Authors
SPDX-License-Identifier: MIT
-->
# 6. SMB2 QUERY_INFO and SET_INFO classes

**Model:** smb2 · **Size:** medium · **Status:** todo

## What it is

QUERY_INFO (MS-SMB2 §2.2.37, §3.3.5.20) and SET_INFO (§2.2.39, §3.3.5.21)
read and change metadata. Each request names an information class. The file
classes are defined in MS-FSCC §2.4, and their behaviour in MS-FSA. Windows
clients use many of them all the time: standard info for size and link count,
network open info on every open, attribute tag info, stream info,
allocation size, and basic info to set times and attributes.

## What the model has today

- **QUERY_INFO:** only `CQueryBasic(FidSel)` (`quint/smb2/smb2_ops.qnt:284`),
  whose reply `RQueryBasic` carries an `AttrSnap` (ftype, nlink, sizeBlocks).
- **SET_INFO:** only three classes, as separate commands:
  - `CSetEof` (`FileEndOfFileInformation`)
  - `CSetDisposition` (`FileDispositionInformation`)
  - `CSetRename` (`FileRenameInformation`)
- No filesystem or security info classes.

## What we'll add

Pick classes by how much behaviour they carry, not by how many there are.
Suggested order:

1. **`FileStandardInformation`** (query). Size, link count, delete-pending,
   directory flag. Delete-pending is the interesting field, because the model
   already tracks delete-on-close.
2. **`FileBasicInformation`** (set). Attributes such as read-only and hidden.
   Timestamps stay abstract. The read-only attribute changes what later opens
   may do, which is real behaviour.
3. **`FileAllocationInformation`** (set). Allocation size versus end of
   file, alongside `CSetEof`.
4. **`FileLinkInformation`** (set). Creates a hard link, with its own
   `ReplaceIfExists`. Pair it with [gap 4](04-smb2-rename-replace.md).
5. **`FileFsSizeInformation` and friends** (filesystem query). Mostly fixed
   values, so they only test the error paths. Low priority.

For each class:
- A model request and reply.
- The access check the class needs. Each class requires certain access
  rights, for example `FILE_WRITE_ATTRIBUTES` for basic info.
- `STATUS_INVALID_INFO_CLASS` and `STATUS_INFO_LENGTH_MISMATCH` arms.
- Tests, and wire support in `harness/samba/smb2_wire.py`.

## Open questions

- Should these become one generic `CQueryInfo({ cls, ... })` and
  `CSetInfo`, or keep one command per class as now? Generic is closer to the
  wire; per-class is closer to the current model style. Ask the maintainers.
- How much does smbprotocol already decode? It has info-class structures for
  many classes, which saves harness work.

## Done when

- The chosen classes have tests and replay against Samba, and any divergence
  is recorded.
