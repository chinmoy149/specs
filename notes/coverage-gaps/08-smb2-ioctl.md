<!--
SPDX-FileCopyrightText: 2026 The Quint Specs Authors
SPDX-License-Identifier: MIT
-->
# 8. SMB2 IOCTL / FSCTLs

**Model:** smb2 · **Size:** large · **Status:** todo

## What it is

IOCTL (MS-SMB2 §2.2.31, §3.3.5.15) carries a control code. The ones a file
server handles are FSCTLs, defined in MS-FSCC §2.3 with behaviour in MS-FSA.
It's a family of unrelated operations sharing one command. The ones that
matter for a NAS:

| FSCTL | What it does |
|-------|--------------|
| `FSCTL_SRV_COPYCHUNK` / `_WRITE` | Server-side copy (MS-SMB2 §3.3.5.15.6). Used by Windows Explorer copies. |
| `FSCTL_SET_SPARSE` | Mark a file sparse. |
| `FSCTL_SET_ZERO_DATA` | Punch a hole. |
| `FSCTL_QUERY_ALLOCATED_RANGES` | List the non-hole ranges. |
| `FSCTL_DUPLICATE_EXTENTS_TO_FILE` | Reflink or clone. |
| `FSCTL_VALIDATE_NEGOTIATE_INFO` | Protocol security check. Out of scope with auth. |
| `FSCTL_GET_REPARSE_POINT` / `SET_` | Symlinks and reparse data. |

## What the model has today

- Nothing. No IOCTL in `Cmd` (`quint/smb2/smb2_ops.qnt:254`) and no mention
  in `quint/smb2/`.
- Related behaviour exists elsewhere in the repo:
  - nfs4 models holes (SEEK, ALLOCATE, DEALLOCATE).
  - nfs4 models COPY and CLONE.
  - The posix model has `RFallocate`, `RCopyRange` and `RCloneRange`.

  The fs core smb2 shares with nfs4 (`quint/nfs/nfs4_fs.qnt`) may already
  carry the hole state.

## What we'll add

Do one FSCTL per PR. Suggested first ones:

**8a: sparse files.** `FSCTL_SET_SPARSE`, `FSCTL_SET_ZERO_DATA`,
`FSCTL_QUERY_ALLOCATED_RANGES`. They fit together: mark sparse, punch a
hole, read back the ranges. Model them on what nfs4 does for DEALLOCATE and
SEEK.
- Errors:
  - zero data on a directory
  - a handle without write access
  - a range with `begin > end`
  - an output buffer too small for the ranges (`BUFFER_OVERFLOW` with partial
    results)

**8b: server-side copy.** `FSCTL_SRV_REQUEST_RESUME_KEY` on the source, then
`FSCTL_SRV_COPYCHUNK` on the target. Copy-chunk has server limits on chunk
count and size, reported back on `INVALID_PARAMETER`. Those limits are
deployment values, probably policy constants.

For each: a generic `CIoctl({ fid, code, ... })`, or one command per FSCTL.
Plus tests, and harness support.

## Open questions

- smbprotocol has an `ioctl` module. How much of these FSCTLs does it encode?
  That decides how big the harness work is.
- Does the shared fs core track allocated ranges, or only sizes? If only
  sizes, 8a starts with an fs-core change, which also affects nfs4.

## Done when

- 8a is modelled, tested and replays against Samba. 8b and the rest are
  tracked as follow-ups.
