# flexfiles-v2-delta-writes

Work-in-progress companion Internet-Draft to
[`draft-haynes-nfsv4-flexfiles-v2`](https://github.com/ietf-wg-nfsv4/flexfiles-v2)
defining the delta-write extension (`CHUNK_XOR_DELTA`).

## Status

**Pre-submission.** The draft source is
[`draft-haynes-nfsv4-flexfiles-v2-delta-writes.md`](draft-haynes-nfsv4-flexfiles-v2-delta-writes.md).
It has not yet been submitted to the datatracker. Ongoing design
iteration happens in this repository.

The build uses Martin Thomson's i-d-template: run `make` to produce
`.txt` and `.html`; `make idnits` to run `idnits` against the rendered
output.

## Scope

The base FFv2 chunk protocol
([`draft-haynes-nfsv4-flexfiles-v2-chunks`](https://github.com/ietf-wg-nfsv4/flexfiles-v2))
makes `CHUNK_WRITE` the sole client-issued data write, and every
`CHUNK_WRITE` carries a **full chunk payload**. For small edits inside
larger chunks on an erasure-coded layout, that forces client-side
stripe fetch, re-encode, and transmit on every edit -- a wire
amplification of three to four orders of magnitude per byte edited.
For a 16-byte edit inside a 256 KiB stripe, the base path costs
roughly 256 KiB of fetch plus 384 KiB of transmit per writer.

When the encoding is XOR-based this is avoidable. If the client
computes

    D = D_old XOR D_new

and the parity encoding is expressible as an XOR combination of source
bytes, then updating any parity projection reduces to XORing the delta
into a specific offset of the stored projection. The same 16-byte edit
on a k=4 m=2 layout drops to roughly 96 bytes across the six
projection data servers.

This draft defines **`CHUNK_XOR_DELTA`** (operation 100): the client
transmits a per-projection XOR delta directly to each data server
holding a projection of the affected stripe, and the data server
applies it locally. It reuses the existing chunk state machine with
**no new commit protocol**.

Applicability is deliberately bounded to **XOR-linear systematic
encodings** and **XOR-affine checksums**; the draft covers
checksum-homomorphism and envelope handling, delta epochs and
per-chunk log state (including size bounds, overflow, retention, and
GC), concurrency and split-open recovery, interaction with
`CHUNK_FINALIZE`/`CHUNK_ROLLBACK` and the repair path, and layout
revocation/stateid semantics.

The motivating workload is the "multiple writers, disjoint regions"
class from
[`draft-haynes-nfsv4-flexfiles-v2-requirements`](https://github.com/ietf-wg-nfsv4/flexfiles-v2):
HPC checkpointing where thousands of ranks write disjoint regions of
the same file in lockstep. The draft includes a worked example at 1000
ranks.

## Relation to the main draft

- This draft normatively depends on the FFv2 requirements, chunks,
  encoding-registry, Mojette, and trust-stateid drafts.
- The extension is **optional**; a server that does not implement it
  continues to serve the base full-chunk `CHUNK_WRITE` path.

## License

AGPL-3.0-or-later inherits from the main repository's practice;
prose contributions are under the IETF trust licensing terms once
an Internet-Draft is submitted.

## Contact

loghyr@gmail.com. Protocol discussion happens on the NFSv4 WG list
(nfsv4@ietf.org) once the work is submitted; until then, open an
issue in this repository.
