<!--
SPDX-FileCopyrightText: 2026 The Quint Specs Authors
SPDX-License-Identifier: MIT
-->
# 9. NFSv4 operations not in the model

**Model:** nfs4 · **Size:** varies · **Status:** todo

## What it is

The nfs4 model has 59 reply kinds (the `S...` variants in
`quint/nfs/nfs4_ops.qnt`). These operations from RFC 7530, RFC 8881 and
RFC 7862 are not among them. TEST_STATEID is also missing, but it has its own
doc: [gap 1](01-nfs4-test-stateid.md).

| Operation | RFC | Worth adding? | Notes |
|-----------|-----|---------------|-------|
| SECINFO_NO_NAME | 8881 §18.45 | **yes** | SECINFO is modelled (`opSecinfo`, `nfs4_ops.qnt:1256`); this is its 4.1 partner for the current or parent handle. Small. |
| OFFLOAD_STATUS, OFFLOAD_CANCEL | 7862 §15.9, §15.8 | **yes** | Asynchronous COPY. The model's COPY is synchronous only (`opCopy`, `nfs4_ops.qnt:3395`), so an async COPY mode comes first. Medium rather than small. |
| COPY_NOTIFY | 7862 §15.3 | maybe | Inter-server copy. Needs a second server; probably out of scope. |
| GETDEVICELIST | 8881 §18.41 | maybe | GETDEVICEINFO is modelled. Small, but rarely used by clients. |
| LAYOUTERROR, LAYOUTSTATS | 7862 §15.6, §15.7 | maybe | pNFS reporting. Mostly "accept and ignore", so few interesting outcomes. |
| WANT_DELEGATION | 8881 §18.49 | maybe | Asks for a delegation without an OPEN. Interacts with recall, which the model barely has. |
| PUTPUBFH | 7530 §16.23 | small | Like PUTROOTFH, for the public filehandle. Few outcomes. |
| OPENATTR | 7530 §16.17 | low | Named attributes; rarely supported. `NOTSUPP` is a fine answer. |
| DELEGPURGE | 7530 §16.5 | low | Only meaningful with reclaim; see [gap 10](10-nfs4-reclaim.md). |
| BACKCHANNEL_CTL | 8881 §18.33 | low | Needs a modelled back channel. |
| SET_SSV | 8881 §18.47 | no | SSV security; out of scope with authentication. |

**The callback channel is missing as a whole.** No server-to-client
operation (CB_RECALL, CB_LAYOUTRECALL, CB_NOTIFY, CB_RECALL_ANY, CB_OFFLOAD,
CB_GETATTR) is modelled as a message. The model records that a recall
happened, but not the callback itself. That's a large piece of work of its
own, and it's what WANT_DELEGATION, OFFLOAD and BACKCHANNEL_CTL eventually
need.

## Harness note

`harness/nfs/nfs4_client.py` defines opcodes only for the operations the
model generates. Each operation above needs its opcode, an encoder and a
decoder added there, and an arm in `nfs4_replay.py`.

## What we'll add

Take the "yes" rows first, one PR each, with the same five steps as every gap
(see the [tracker](README.md)).

## Open questions

- An async COPY returns a callback (CB_OFFLOAD) when it finishes. Can
  OFFLOAD_STATUS polling stand in for it until the callback channel exists?
- For the low-value rows, is "the server answers `NOTSUPP` or `OP_ILLEGAL`
  correctly" worth a model arm? It's cheap and catches servers that crash on
  unknown ops.

## Done when

- SECINFO_NO_NAME and the OFFLOAD pair are in, or have been ruled out with a
  reason written here.
