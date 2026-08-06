---
title: Delta-Write Extension for the Flexible File Version 2 Layout Type
abbrev: FFv2 Delta Writes
docname: draft-haynes-nfsv4-flexfiles-v2-delta-writes-latest
category: std
date: {DATE}
consensus: true
ipr: trust200902
area: General
workgroup: Network File System Version 4
keyword: [pNFS, flexfiles, erasure coding, Mojette, XOR, HPC]

stand_alone: yes
pi: [toc, sortrefs, symrefs, docmapping, comments]

venue:
  group: Network File System Version 4
  type: Working Group
  mail: nfsv4@ietf.org
  arch: https://mailarchive.ietf.org/arch/browse/nfsv4/
  github: ietf-wg-nfsv4/flexfiles-v2-delta-writes
  latest: https://ietf-wg-nfsv4.github.io/flexfiles-v2-delta-writes/draft-haynes-nfsv4-flexfiles-v2-delta-writes.html

author:
 -
    ins: T. Haynes
    name: Thomas Haynes
    organization: Hammerspace
    email: loghyr@gmail.com

normative:
  RFC4506:
  RFC5661:
  I-D.haynes-nfsv4-flexfiles-v2:

informative:
  MOJETTE-1995:
    title: "The Mojette Transform: Application to Image Coding"
    author:
      - name: J-P. Guédon
      - name: N. Normand
    date: 1995

--- abstract

The Flexible File Version 2 pNFS layout type
{{I-D.haynes-nfsv4-flexfiles-v2}} defines a chunk-oriented
data-server protocol in which every write is a full-chunk payload.
For workloads that make small edits to files protected by an
XOR-based erasure encoding, this forces client-side stripe fetch,
re-encode, and transmit on every edit, with wire amplification of
three to four orders of magnitude per byte edited.  This document
defines an optional extension, CHUNK_XOR_DELTA, that lets a client
transmit a per-projection XOR delta directly to each data server
holding a projection of the affected stripe; the data server
applies the delta locally.  The extension is restricted to
XOR-linear systematic encodings and XOR-affine checksums, using
the existing chunk state machine with no new commit protocol.

--- middle

# Introduction {#sec-introduction}

The base Flexible File Version 2 specification
{{I-D.haynes-nfsv4-flexfiles-v2}} defines the CHUNK_WRITE operation
as the sole client-issued data-write operation against a data server.
Each CHUNK_WRITE carries a full chunk payload -- either a block
(for mirrored layouts) or a shard (for erasure-coded layouts) --
which the data server places in the PENDING state and later transitions
to FINALIZED and COMMITTED through the operations of the chunk state
machine defined in {{I-D.haynes-nfsv4-flexfiles-v2}}.

For workloads with the following combination of properties, this
model has an unavoidable wire-amplification cost:

- Small edits (bytes to kilobytes) inside larger chunks (typically
  tens of kilobytes to megabytes)
- Erasure-coded layouts where the parity value depends on the
  edited data-shard byte
- Multiple concurrent writers making disjoint edits to the same file,
  such that per-writer fetch-modify-writeback of full stripes creates
  a bandwidth bottleneck disproportionate to the logical bytes edited

The paradigmatic example is the "Multiple writers, disjoint regions
(rare)" workload class named in the Use Cases section of
{{I-D.haynes-nfsv4-flexfiles-v2}}: high-performance
computing (HPC) checkpoint workloads in which thousands of ranks
write disjoint regions of the same file in lockstep.  For a 16-byte
edit inside a 256 KiB stripe, the base CHUNK_WRITE path costs
approximately 256 KiB of stripe fetch plus 384 KiB of
new-stripe-plus-parity transmit per writer per checkpoint interval
-- an amplification of roughly 4x10^4 over the logical edited bytes.

When the erasure encoding is XOR-based, this amplification is
avoidable.  If the client can compute the delta

    D = D_old XOR D_new

between the pre-edit and post-edit values of the affected bytes,
and the parity encoding is expressible as an XOR combination of
source bytes, then updating any parity projection reduces to XORing
the delta into a specific offset of the stored projection.  For a
16-byte edit on a k=4 m=2 layout, this reduces the wire cost from
approximately 256 KiB to roughly 96 bytes across the six projection
data servers.

This document defines CHUNK_XOR_DELTA, an optional operation that
transmits per-projection deltas.  Applicability is bounded by two
independent capability flags, both derived from static properties of
the encoding-plus-checksum pair the layout already carries:

- The erasure encoding is XOR-linear in its parity computation AND
  systematic (D_old for any byte range is directly readable from a
  single projection).  Registry flag EC_ENC_FLAGS_XOR_DELTA_CAPABLE,
  this document.  FFV2_ENCODING_MIRRORED (identity encoding,
  degenerate case), FFV2_ENCODING_MOJETTE_SYSTEMATIC, and
  FFV2_ENCODING_XOR_PARITY qualify.
  FFV2_ENCODING_MOJETTE_NON_SYSTEMATIC is XOR-linear but not
  systematic; see {{sec-scope}} for why it is excluded.
- The chunk envelope's checksum algorithm is XOR-affine (registry
  flag CHECKSUM_FLAGS_XOR_AFFINE, this document); the CRC family
  qualifies, cryptographic hashes and modular-sum checksums do not.

The client determines capability for a given layout by looking up
the layout's declared encoding in the Erasure Encoding Type
Registry and its `ffv2m_checksum_algorithm` in the Checksum
Algorithm Registry, both established in the IANA Considerations of
{{I-D.haynes-nfsv4-flexfiles-v2}}.  When both
registry flags are set the client MAY issue CHUNK_XOR_DELTA against
that layout; when either is clear it MUST NOT.  No new field is
added to `ffv2_mirror4`; capability is fully derivable from fields
already present.

## Requirements Language

{::boilerplate bcp14-tagged}

## Relationship to Base Specification

This document extends the flexible file v2 layout protocol family with
one new operation (CHUNK_XOR_DELTA), one new error code
(NFS4ERR_DELTA_INCOMPLETE), one new advisory-warning code
(NFS4ERR_DELTA_LOG_FULL), and additions to two IANA registries
established by {{I-D.haynes-nfsv4-flexfiles-v2}}: the Checksum
Algorithm Registry and the Erasure Encoding Type Registry.  All
mechanisms defined here reuse the chunk state machine, chunk_guard4
CAS primitive, repair protocol, and layout-revocation paths defined in
{{I-D.haynes-nfsv4-flexfiles-v2}}.

# Terminology {#sec-terminology}

The terms block, shard, chunk, chunk state machine, chunk
generation, chunk owner, and projection are defined in the base
specification.  This document uses them without redefinition.

Additional terms defined by this document:

delta:

: A byte sequence D such that D = D_old XOR D_new, where D_old is the
  current value of a contiguous range of bytes within a chunk and
  D_new is the client-intended replacement value.  A delta is applied
  to a stored chunk by XORing D into the same byte range.

delta epoch:

: A contiguous sequence of CHUNK_XOR_DELTA operations issued by a
  single client against a single chunk, bracketed by a chunk_guard4
  CAS at open time and a CHUNK_FINALIZE at close time.  All deltas
  within an epoch share the same chunk_guard4 generation and are
  ordered by a monotonic sequence number assigned by the client.

delta log:

: The per-chunk, in-data-server record of deltas received during an
  active delta epoch.  Each log entry records (sequence number, byte
  offset, delta bytes).  The log is bounded in size and self-inverse:
  applying an entry a second time undoes it.

XOR-linear encoding:

: An erasure encoding whose parity computation can be expressed as
  a linear combination in GF(2) of source bytes -- equivalently, an
  encoding where changing one source byte by delta D and applying
  the same D to each parity projection at the projection-specific
  offset preserves encode-correctness.  FFV2_ENCODING_MIRRORED (as
  the degenerate identity-encoding case; every mirror is a
  byte-identical replica), FFV2_ENCODING_MOJETTE_SYSTEMATIC (defined in
  the Mojette Transform Encoding section of
  {{I-D.haynes-nfsv4-flexfiles-v2}}; the underlying discrete
  Radon transform is due to {{MOJETTE-1995}}), and
  FFV2_ENCODING_XOR_PARITY are all XOR-linear AND systematic
  (D_old readable from a single projection).
  FFV2_ENCODING_MOJETTE_NON_SYSTEMATIC is XOR-linear but not
  systematic: recovering D_old requires reading k projections and
  inverting the transform.  FFV2_ENCODING_RS_VANDERMONDE is not
  XOR-linear at all (it is linear over GF(2^8), which requires
  per-coefficient multiplication that XOR alone does not express).
  Only the XOR-linear AND systematic subset qualifies for
  CHUNK_XOR_DELTA under this document.

XOR-affine checksum:

: A checksum algorithm f such that for any two byte sequences X and
  Y of equal length L, f(X XOR Y) = f(X) XOR f(Y) XOR f(0^L), where
  0^L is the L-byte all-zero sequence.  Equivalently, the "raw"
  form of f with init/xorout constants set to zero satisfies the
  homomorphism f_raw(X XOR Y) = f_raw(X) XOR f_raw(Y).  Standard
  CRC32 and CRC32C (as deployed with init and xorout both
  0xFFFFFFFF) satisfy the affine identity above but do not
  satisfy the stricter homomorphism f(X XOR Y) = f(X) XOR f(Y); the
  affine constant f(0^L) is a function of length alone, so any two
  implementations agreeing on the algorithm and the operand length
  agree on the result.  All incremental formulas in this document
  are expressed in terms that make the affine correction explicit
  (see {{sec-checksum}}).  CHECKSUM_ALG_CRC32 and
  CHECKSUM_ALG_CRC32C qualify.  Cryptographic hashes
  (CHECKSUM_ALG_SHA256, CHECKSUM_ALG_SHA512, CHECKSUM_ALG_BLAKE3)
  and CHECKSUM_ALG_FLETCHER4 (a modular-sum checksum) do not
  qualify under any construction.

# Encoding-Family Scope {#sec-scope}

CHUNK_XOR_DELTA MAY be used against a chunk if and only if both of
the following hold, as determined by registry lookup against the
chunk's governing layout:

- The layout's declared encoding (see the base specification's
  `ffv2_coding_type4` and the Erasure Encoding Type Registry in
  {{I-D.haynes-nfsv4-flexfiles-v2}}) has the flag
  EC_ENC_FLAGS_XOR_DELTA_CAPABLE set ({{sec-iana-encoding-flag}});
  i.e., the encoding is XOR-linear as defined in
  {{sec-terminology}}.
- The layout's `ffv2m_checksum_algorithm` has the flag
  CHECKSUM_FLAGS_XOR_AFFINE set in the Checksum Algorithm
  Registry defined in {{I-D.haynes-nfsv4-flexfiles-v2}}
  ({{sec-iana-checksum-flag}}).

Capability is thus a derived, static property of the (encoding,
checksum-algorithm) pair the layout already carries; no additional
field is added to `ffv2_mirror4`.  A data server MUST reject
CHUNK_XOR_DELTA against a chunk whose governing layout does not
satisfy both conditions with NFS4ERR_NOTSUPP.  A client SHOULD perform
the registry lookup at layout-grant time and cache the result for the
layout's lifetime; the data server MUST perform the check on each
operation received (the data server cannot assume clients have
honored the SHOULD).

Encodings that register EC_ENC_FLAGS_XOR_DELTA_CAPABLE MUST specify,
in the document defining the encoding, the mapping from
`(chunk_offset, byte_offset_within_chunk)` to the projection-local
offset at which a delta is XORed.  For
FFV2_ENCODING_MOJETTE_SYSTEMATIC this mapping is defined in the
Mojette Transform Encoding section of
{{I-D.haynes-nfsv4-flexfiles-v2}}.  For
FFV2_ENCODING_XOR_PARITY the mapping is trivial: for the parity
projection the delta is XORed at the same offset as it appears in
the source chunk.  For FFV2_ENCODING_MIRRORED the mapping is also
trivial: on each mirror the delta is XORed at the same offset as in
the source chunk.  FFV2_ENCODING_MOJETTE_NON_SYSTEMATIC is
XOR-linear but does not register the flag, because computing D_old
requires reading and inverse-transforming k projections, which is
the read-modify-write cost this document exists to eliminate.

This document adds the EC_ENC_FLAGS_XOR_DELTA_CAPABLE flag to the
encoding registry.  Encodings that do not qualify
(FFV2_ENCODING_RS_VANDERMONDE; also FFV2_ENCODING_LINUX_MD_RAID for
its Q shard) MAY be extended by a separate document defining a
GF-multiply-per-parity variant of the delta-write operation; that
extension is out of scope here.

# Operation 100: CHUNK_XOR_DELTA - Apply XOR Delta to Stored Chunk {#sec-CHUNK_XOR_DELTA}

## OPERATION NUMBER AND DISPATCH

Following the pattern in
{{I-D.haynes-nfsv4-flexfiles-v2}} for CHUNK operations, this
document allocates operation number 100 and adds
corresponding arms to the argument and result unions.  All XDR
definitions in this document use the language of {{RFC4506}}.

~~~ xdr
   /// const OP_CHUNK_XOR_DELTA = 100;
~~~
{: #fig-OP_CHUNK_XOR_DELTA title="XDR for OP_CHUNK_XOR_DELTA" }

~~~ xdr
   /// case OP_CHUNK_XOR_DELTA: CHUNK_XOR_DELTA4args opchunkxordelta;
~~~
{: #fig-nfs_argop4-arm title="nfs_argop4 amendment arm" }

~~~ xdr
   /// case OP_CHUNK_XOR_DELTA: CHUNK_XOR_DELTA4res opchunkxordelta;
~~~
{: #fig-nfs_resop4-arm title="nfs_resop4 amendment arm" }

## ARGUMENTS

~~~ xdr
   /// const CHUNK_XOR_DELTA_MAX_ENTRIES  = 8;
   /// const CHUNK_XOR_DELTA_MAX_DELTA_LEN = 65536;
~~~
{: #fig-chunk_xor_delta_bounds title="Wire-size bounds" }

~~~ xdr
   /// struct chunk_xor_delta_entry4 {
   ///     uint32_t   cxde_seq;
   ///     uint32_t   cxde_bin_offset;
   ///     opaque     cxde_delta<CHUNK_XOR_DELTA_MAX_DELTA_LEN>;
   /// };
~~~
{: #fig-chunk_xor_delta_entry4 title="XDR for chunk_xor_delta_entry4" }

~~~ xdr
   /// const CHUNK_XOR_DELTA_FLAGS_EPOCH_OPEN     = 0x00000001;
   /// const CHUNK_XOR_DELTA_FLAGS_EPOCH_CONTINUE = 0x00000002;
   /// /* 0x00000004 reserved; formerly EPOCH_CLOSE.  See prose. */
   ///
   /// struct CHUNK_XOR_DELTA4args {
   ///     /* CURRENT_FH: file */
   ///     stateid4                  cxda_stateid;
   ///     offset4                   cxda_chunk_offset;
   ///     chunk_owner4              cxda_owner;
   ///     uint32_t                  cxda_flags;
   ///     chunk_guard4              cxda_guard;
   ///     chunk_guard4              cxda_predecessor_guard;
   ///     chunk_xor_delta_entry4
   ///                        cxda_deltas<CHUNK_XOR_DELTA_MAX_ENTRIES>;
   /// };
~~~
{: #fig-CHUNK_XOR_DELTA4args title="XDR for CHUNK_XOR_DELTA4args" }

## RESULTS

~~~ xdr
   /// struct CHUNK_XOR_DELTA4resok {
   ///     uint32_t          cxdr_high_water_seq;
   ///     uint32_t          cxdr_log_bytes_used;
   ///     uint32_t          cxdr_log_bytes_available;
   /// };
~~~
{: #fig-CHUNK_XOR_DELTA4resok title="XDR for CHUNK_XOR_DELTA4resok" }

~~~ xdr
   /// union CHUNK_XOR_DELTA4res switch (nfsstat4 cxdr_status) {
   ///     case NFS4_OK:
   ///         CHUNK_XOR_DELTA4resok    cxdr_resok4;
   ///     default:
   ///         void;
   /// };
~~~
{: #fig-CHUNK_XOR_DELTA4res title="XDR for CHUNK_XOR_DELTA4res" }

## DESCRIPTION

The CHUNK_XOR_DELTA operation applies one or more XOR deltas to a
single stored chunk on a data server.  It is issued by a client that
holds an active layout for the file, an active stateid, and a valid
chunk_owner4 for the chunk.  The data server MUST reject the operation
with NFS4ERR_NOTSUPP if the governing layout's encoding-plus-checksum
combination is not XOR-delta-capable (see {{sec-scope}}).

The operation targets the chunk identified by `cxda_chunk_offset`.
`cxda_flags` indicates the role of the operation within a delta epoch:

- CHUNK_XOR_DELTA_FLAGS_EPOCH_OPEN opens a new delta epoch.
  `cxda_guard` carries a fresh, client-chosen guard value that
  will become the epoch's identifier if the open succeeds;
  `cxda_predecessor_guard` carries the guard value the client
  believes is currently COMMITTED on this chunk (the value under
  which D_old was read).  The data server MUST:

    (1) Reject with NFS4ERR_CHUNK_GUARDED if any other delta epoch
        (owned by any client) is currently open on this chunk.
    (2) Reject with NFS4ERR_CHUNK_GUARDED if the chunk's current
        COMMITTED guard does not equal `cxda_predecessor_guard` --
        the client's D_old is stale, and applying the delta would
        silently corrupt the parity.
    (3) On success, atomically install `cxda_guard` as the new
        PENDING guard and allocate a fresh delta log for the chunk
        keyed by (cxda_guard, cxda_owner).

  Retransmit handling: if the data server receives an EPOCH_OPEN whose
  (cxda_guard, cxda_owner) exactly matches an already-open epoch it
  owns, the data server MUST treat this as a retransmit of the original
  request and return the original success response, not
  NFS4ERR_CHUNK_GUARDED.  This preserves idempotent replay under
  RPC retransmission (see {{sec-concurrency}}).
- CHUNK_XOR_DELTA_FLAGS_EPOCH_CONTINUE indicates the operation is part
  of an already-open epoch.  The data server MUST verify that the
  epoch identified by (cxda_guard, cxda_owner) is open and that
  `cxda_owner` matches the owner of the open epoch; mismatch is
  rejected with NFS4ERR_CHUNK_GUARDED.  `cxda_predecessor_guard` is
  ignored on EPOCH_CONTINUE operations; the data server SHOULD verify
  it is present in the wire message (per XDR) but MUST NOT use its
  value to gate acceptance.  Exactly one of EPOCH_OPEN or
  EPOCH_CONTINUE MUST be set; setting neither or both MUST be rejected
  with NFS4ERR_INVAL.  Bit value 0x00000004 in cxda_flags is reserved
  (formerly intended as an explicit EPOCH_CLOSE bit; epoch closure is
  instead signalled by a subsequent CHUNK_FINALIZE, see
  {{sec-state-machine}}).  Any other bit set in cxda_flags MUST be
  rejected with NFS4ERR_INVAL to allow forward-compatible flag
  additions.

`cxda_deltas` is a bounded array of delta entries; the wire XDR bounds
the array at 8 entries per operation, and the sum of `cxde_delta`
lengths across all entries in a single operation MUST NOT exceed 65536
bytes.  A data server SHOULD reject an operation exceeding either
limit with NFS4ERR_INVAL rather than accepting a truncated set.

Each entry carries a monotonic sequence number `cxde_seq`
(client-chosen, MUST strictly increase across all CHUNK_XOR_DELTA
operations
within an epoch), a byte offset `cxde_bin_offset` within the chunk,
and the delta bytes themselves.  The data server applies each entry by
XORing `cxde_delta` into stored-chunk bytes `[cxde_bin_offset,
cxde_bin_offset + len(cxde_delta))`.  Delta entries within a single
operation MAY be applied by the data server in any order, since XOR is
commutative and associative; the sequence number is recorded per entry
for later completeness checking at CHUNK_FINALIZE time.

The data server MUST reject with NFS4ERR_INVAL any entry whose byte
range extends beyond the chunk's declared size, or whose
`cxde_bin_offset` overlaps another entry's byte range within the same
operation (to preserve deterministic replay across implementations).
Cross- operation range overlap within an epoch is permitted -- the
second delta effectively updates the running XOR state.

## RETURN VALUES

On success the data server returns:

- `cxdr_high_water_seq`: the highest sequence number the data server
  has seen in this epoch, including all entries in the current
  operation.
- `cxdr_log_bytes_used`: the current delta-log occupancy for this
  chunk-epoch, in bytes.
- `cxdr_log_bytes_available`: the log's remaining capacity, in bytes.
  When this reaches zero, subsequent CHUNK_XOR_DELTA operations in
  this epoch MUST be rejected with NFS4ERR_DELTA_LOG_FULL and the
  client MUST close the epoch with CHUNK_FINALIZE or CHUNK_ROLLBACK
  before starting a new one.

On failure the data server returns the appropriate nfsstat4 code and no
partial application is visible via CHUNK_READ; see
{{sec-state-machine}} for the visibility rules.

# Checksum-Homomorphism and Envelope Handling {#sec-checksum}

Each chunk carries an envelope that includes a checksum computed
over the sequence `chunk_header || chunk_data`, as defined in
{{I-D.haynes-nfsv4-flexfiles-v2}}.  A delta write modifies
both parts of that sequence:

- The chunk_data portion changes by the applied delta (byte-range
  XOR at `cxde_bin_offset`).
- The chunk_header portion changes because CHUNK_FINALIZE assigns a
  new chunk generation identifier to the finalized chunk, updating
  header fields.

The checksum algorithm registry defined by
{{I-D.haynes-nfsv4-flexfiles-v2}} MUST be extended with a
boolean capability flag CHECKSUM_FLAGS_XOR_AFFINE
({{sec-iana-checksum-flag}}).  CHECKSUM_ALG_CRC32 and
CHECKSUM_ALG_CRC32C set this flag; CHECKSUM_ALG_FLETCHER4 and the
cryptographic-hash algorithms (CHECKSUM_ALG_SHA256,
CHECKSUM_ALG_SHA512, CHECKSUM_ALG_BLAKE3) do not.

At CHUNK_FINALIZE time -- not per CHUNK_XOR_DELTA -- the data server
is responsible for computing the new envelope checksum.  For an
XOR-affine checksum (see the terminology definition in
{{sec-terminology}} for the exact identity), the data server MAY
compute the new checksum incrementally.  Let L be the length of the
covered envelope (chunk_header || chunk_data), let X be the pre-delta
envelope contents, and let Y be the post-delta envelope contents
zero-extended to length L in the same layout.  The affine identity
gives:

    f(Y) = f(X) XOR f(X XOR Y) XOR f(0^L)

where (X XOR Y) is the L-byte sequence containing zeros everywhere
X and Y agree, and the applied header + data delta bytes at their
canonical byte offsets everywhere they differ.  Equivalently, using
the zero-initialized "raw" form of f (per {{sec-terminology}}):

    f_raw(Y) = f_raw(X) XOR f_raw(X XOR Y)

The `f(0^L)` term is a length-only constant that any two
conforming implementations agreeing on the algorithm and the
covered length compute identically.

Full-recomputation is also permitted and is required for algorithms
that do not implement the incremental combine.  In either case the
client does not supply the envelope checksum; the data server is the
sole computing authority.  Implementations MUST NOT combine partial
checksums across differing covered lengths; the affine correction term
is a function of length and combining across differing lengths
silently yields the wrong value.

At CHUNK_FINALIZE time the data server MUST include the newly computed
envelope checksum in its response, in the same field
{{I-D.haynes-nfsv4-flexfiles-v2}} uses for CHUNK_WRITE-driven
finalization.  A client that participated in the epoch SHOULD verify
this against its own predicted post-delta checksum computed from D_old
and the delta sequence (using the same affine identity above); a
client that performs this verification MUST treat a mismatch as chunk
corruption (the same treatment applied to a CHUNK_READ checksum
mismatch on COMMITTED data).  End-to-end verification is SHOULD rather
than MUST because the data server's checksum is itself protected by
the base checksum-registry semantics; the client-side check adds a
second, independent detector for lost-or-corrupt-delta cases that the
data server side alone cannot detect.

# Delta Epochs and Per-Chunk State {#sec-epochs}

A delta epoch is the unit of delta-write atomicity.  An epoch is
opened by a CHUNK_XOR_DELTA with EPOCH_OPEN set; extended by zero or
more CHUNK_XOR_DELTA operations with EPOCH_CONTINUE set; and closed
by a CHUNK_FINALIZE on the same chunk (or aborted by CHUNK_ROLLBACK).

At most one epoch is open on a chunk at any time.  Attempting to
open a second epoch while one is already open on the same chunk MUST
be rejected with NFS4ERR_CHUNK_GUARDED, regardless of whether the
requesting client owns the open epoch: the tiebreaker is the CAS
value in `cxda_guard`.  This constraint is consistent with the base
specification's single-writer-per-chunk model.

## Delta Log Structure

For each open epoch the data server maintains a delta log recording
every applied delta entry.  Each log record contains at minimum:

- The delta entry's `cxde_seq`
- The delta entry's `cxde_bin_offset` and length
- The delta bytes themselves

Because XOR entries are self-inverse (applying an entry a second time
undoes it), the same log serves as both the redo log (for restart
after data server crash mid-epoch) and the undo log (for
CHUNK_ROLLBACK).

## Log Size Bound and Overflow

The data server MUST bound the per-chunk delta log to a fixed maximum
size, at minimum:

    max(4096, min(chunk_size / 4, 65536)) bytes

A conforming data server MAY implement a larger bound.  The data
server reports current occupancy and remaining capacity in every
CHUNK_XOR_DELTA response (`cxdr_log_bytes_used`,
`cxdr_log_bytes_available`).

When a client issues a CHUNK_XOR_DELTA whose acceptance would exceed
the bound, the data server MUST return NFS4ERR_DELTA_LOG_FULL and MUST
NOT apply any entry in that operation (all-or-nothing per operation).
The client's options on receiving NFS4ERR_DELTA_LOG_FULL are:

- Close the current epoch with CHUNK_FINALIZE and open a new one
  with the next CHUNK_XOR_DELTA (naturally amortizing the log across
  epochs)
- Abort the current epoch with CHUNK_ROLLBACK and fall back to
  CHUNK_WRITE for the remaining edits in this checkpoint interval

## Log Retention and Garbage Collection

The delta log for an epoch is retained by the data server until the
epoch is either committed (CHUNK_FINALIZE followed by CHUNK_COMMIT) or
aborted (CHUNK_ROLLBACK).  On commit, the log is discarded once the
data server has transitioned the chunk to COMMITTED with the new
generation; on abort, the log is discarded immediately after the data
server has XORed every log entry back into the chunk (undo).

The chunk-generation retention rule defined in
{{I-D.haynes-nfsv4-flexfiles-v2}} -- that a data server retains the
last COMMITTED generation of each chunk until superseded by a newer
COMMIT -- applies unchanged.  The delta log is auxiliary state that
lives alongside the PENDING/FINALIZED generation being built up, not
a replacement for that state.

# Concurrency Semantics {#sec-concurrency}

Within a single open epoch the delta log establishes a total order
over applied entries by `cxde_seq`, but the effect of applied
entries at any point in the epoch is order-independent (XOR is
commutative).  The sequence number exists to:

- Detect gaps at CHUNK_FINALIZE time (the data server asserts
  contiguity from 1 to `cxdr_high_water_seq`)
- Support idempotent replay across RPC retransmissions (a data server
  receiving a duplicate `cxde_seq` MUST NOT re-apply the delta;
  it MUST return the same result as the original response)

The data server is NOT required to maintain per-byte or per-bin lock
state.  The single-writer-per-epoch invariant (enforced by the CAS on
EPOCH_OPEN) is sufficient to prevent conflicting concurrent writes.
Cross-client concurrent edits to disjoint byte ranges of the same
chunk are NOT supported in this specification; the second client
receives NFS4ERR_CHUNK_GUARDED on its EPOCH_OPEN and MUST fall back to
CHUNK_WRITE.  Future extensions MAY relax this constraint by
introducing per-bin versioning; that machinery is not required for the
HPC checkpoint workload, whose block-alignment discipline (base spec
Use Cases section) already gives stable per-chunk ownership within a
checkpoint interval.

## Split-Open Recovery {#sec-split-open}

Because a client opens the epoch independently against each of the
chunk's k+m projection data servers, two clients A and B racing for the
same chunk can produce a split-open outcome: A wins EPOCH_OPEN on
some projections, B wins on others.  Neither can then close its
epoch on the full projection set, and both sets of deltas remain
in invisible PENDING state indefinitely -- blocking not only
subsequent writes to that chunk but also repair (see {{sec-repair}}).

To prevent this liveness hazard: on receiving NFS4ERR_CHUNK_GUARDED
from ANY projection's EPOCH_OPEN, a client MUST issue CHUNK_ROLLBACK
against every projection where its own EPOCH_OPEN had succeeded,
before falling back to CHUNK_WRITE.  A client MUST NOT abandon
partially-open epochs.  The rollback is CAS-guarded by the
client's own `cxda_guard` value, so it cannot disturb the winner's
epoch state on projections the winner controls.  A client that
crashes mid-recovery relies on lease-expiry rollback per
{{sec-repair}}.

## Retransmission {#sec-retransmit}

Idempotency across RPC retransmission is achieved by the sequence
number + CAS-guard pair.  A client retrying a CHUNK_XOR_DELTA after
network loss re-uses the same (cxda_guard, cxda_owner, cxde_seq)
tuple; the data server deduplicates using the recorded sequence number
and returns the same response.  Retransmit handling for EPOCH_OPEN
specifically is normatively described in the CHUNK_XOR_DELTA
DESCRIPTION section: a duplicate EPOCH_OPEN whose (cxda_guard,
cxda_owner) matches an already-open epoch MUST be served as a
retransmit, not rejected with NFS4ERR_CHUNK_GUARDED.

# Interaction with the Chunk State Machine {#sec-state-machine}

The Chunk State Machine section of
{{I-D.haynes-nfsv4-flexfiles-v2}} defines the chunk state
machine with three main states -- PENDING, FINALIZED, COMMITTED --
and the operations that transition between them.  This document adds
no new states and no new transitions.  It defines CHUNK_XOR_DELTA as
a third producer of PENDING generations, alongside CHUNK_WRITE and
CHUNK_WRITE_REPAIR.

## Visibility Rules

The rule from {{I-D.haynes-nfsv4-flexfiles-v2}} that
CHUNK_READ serves the most recent COMMITTED generation applies
without modification.  In particular:

- Deltas applied during an open epoch are NOT visible to CHUNK_READ
  until the epoch has been closed by CHUNK_FINALIZE + CHUNK_COMMIT.
- During an open epoch the data server retains the prior COMMITTED chunk
  contents (already required by
  {{I-D.haynes-nfsv4-flexfiles-v2}} for concurrent-reader
  consistency).  CHUNK_READ served from that state is unchanged by
  any number of applied deltas.

## CHUNK_FINALIZE Semantics for a Delta Epoch

When the client issues CHUNK_FINALIZE against a chunk that has an
open delta epoch, the data server MUST:

- Verify that the recorded delta-log sequence numbers form a
  contiguous range from 1 to some N (no gaps).  If gaps are present,
  reject with NFS4ERR_DELTA_INCOMPLETE and DO NOT discard the log;
  the client MAY retry the missing deltas and re-attempt
  CHUNK_FINALIZE (see {{sec-gap-recovery}}).
- Compute the new envelope checksum per {{sec-checksum}}
- Assign the new chunk generation identifier
- Transition the chunk to FINALIZED
- Retain the delta log until CHUNK_COMMIT completes; on
  CHUNK_ROLLBACK, undo the deltas by re-applying them (XOR
  self-inverse) and discard the log.

### Gap Recovery on NFS4ERR_DELTA_INCOMPLETE {#sec-gap-recovery}

The CHUNK_FINALIZE result union defined in
{{I-D.haynes-nfsv4-flexfiles-v2}} does not carry a
missing-seq array on error return, so the client cannot directly
enumerate which sequence numbers the data server is missing.  Until a
future revision of the CHUNK_FINALIZE result union provides such
enumeration, the client MUST implement gap recovery as follows:

1. The client MUST retain a per-epoch "outstanding" set: every
   `(cxda_guard, cxde_seq)` for which it has issued
   CHUNK_XOR_DELTA but not yet received a success response.
2. On NFS4ERR_DELTA_INCOMPLETE from CHUNK_FINALIZE, the client
   MUST re-issue every `cxde_seq` still in its outstanding set as
   an EPOCH_CONTINUE operation.  Duplicates already seen by the
   data server are deduplicated per {{sec-concurrency}} (return the
   original response with no re-apply); genuine gaps are filled.
3. Once the outstanding set is empty, the client MUST re-issue
   CHUNK_FINALIZE.  If the data server still returns
   NFS4ERR_DELTA_INCOMPLETE, the client MUST issue CHUNK_ROLLBACK
   and restart the edit sequence as CHUNK_WRITE operations under
   a fresh epoch (the epoch is unrecoverable).

This is O(outstanding-set-size) worst-case wire traffic for gap
recovery, versus O(1) for an enumerated missing-seq array.  In
practice the outstanding set is bounded by the delta-log capacity
divided by the mean delta size (typically low hundreds of
entries).

The CHUNK_COMMIT semantics of
{{I-D.haynes-nfsv4-flexfiles-v2}} apply unchanged: the
FINALIZED generation becomes COMMITTED atomically.

## CHUNK_ROLLBACK Semantics for a Delta Epoch

CHUNK_ROLLBACK against a chunk with an open delta epoch causes the
data server to re-apply every log entry (XORing each into the chunk a
second time), restoring the pre-epoch bytes.  The log is then
discarded and the chunk_guard4 CAS state returns to what it was at
EPOCH_OPEN.

# Repair-Path Interaction {#sec-repair}

The repair protocol defined in
{{I-D.haynes-nfsv4-flexfiles-v2}} coordinates reconstruction
of missing or damaged chunks from surviving projections.  This
document adds one new rule for the repair coordinator:

- Before reconstructing a chunk from the majority of surviving
  projections, the repair coordinator MUST query each surviving data
  server for the presence of an open delta epoch on that chunk.
- If any surviving data server reports an open delta epoch, the repair
  coordinator MUST NOT reconstruct from the majority.  Instead, it
  MUST wait for the epoch to close (CHUNK_FINALIZE or
  CHUNK_ROLLBACK) before beginning reconstruction.
- If the epoch's owner lease has expired, OR the epoch's owning
  stateid has been revoked (per {{I-D.haynes-nfsv4-flexfiles-v2}}),
  the repair coordinator MUST drive CHUNK_ROLLBACK on each
  participating data server (using the CAS guard from the epoch's OPEN
  record) and then proceed with base-specification repair semantics on
  the resulting pre-epoch generation.  Both triggers are wall-clock
  bounded -- lease-expiry by the server's lease-time attribute and
  stateid revocation by the base specification's stateid-revocation
  paths -- so repair cannot stall indefinitely on a wedged writer.

Rationale: if the epoch has partially applied to some but not all
projections, reconstructing from the majority would silently commit
the "unchanged" bytes as authoritative and lose any deltas already
applied.  Waiting for epoch closure (or explicitly rolling it back)
prevents this laundering path.

Steady-state repair -- against chunks with no open delta epoch --
is unchanged from {{I-D.haynes-nfsv4-flexfiles-v2}}.  The
delta-log retention rule guarantees that at any moment either the
pre-epoch generation is intact on every participating data server (open
epoch case) or a common COMMITTED generation is present on the
surviving data servers (steady-state case).

# Layout Revocation and Stateid Semantics {#sec-revocation}

CB_LAYOUTRECALL is defined in {{RFC5661}}; the stateid-revocation
paths (TRUST_STATEID, REVOKE_STATEID, BULK_REVOKE_STATEID) are
defined in {{I-D.haynes-nfsv4-flexfiles-v2}}.  A delta
epoch is bound to the client's active layout and stateid; when either
is revoked mid-epoch, the data server MUST:

- Discard any in-flight CHUNK_XOR_DELTA operations for that
  (stateid, chunk) pair
- Apply CHUNK_ROLLBACK semantics ({{sec-state-machine}}) to close the
  epoch, restoring the pre-epoch generation
- Discard the delta log

A client that receives a revocation notification (via CB_LAYOUTRECALL
or an explicit stateid revocation) MUST assume any epoch it had open
against the affected file has been rolled back and MUST NOT issue
CHUNK_XOR_DELTA against that stateid.  The client MAY reissue the
edit sequence as CHUNK_WRITE operations under a fresh layout and
stateid; the write-retry semantics of
{{I-D.haynes-nfsv4-flexfiles-v2}} apply.

This resolves the "layout recalled mid-delta" failure mode without
introducing a new commit protocol: the revocation paths of {{RFC5661}}
and {{I-D.haynes-nfsv4-flexfiles-v2}} already tear
down the client's authority to write, and the data server's duty on
revocation is to preserve the last-committed state -- which is
exactly what pre-epoch rollback delivers.

# Security Considerations {#sec-security}

## Authorization Equivalence

Any principal authorized to issue CHUNK_WRITE against a chunk is,
by construction, authorized to write any byte sequence into that
chunk.  CHUNK_XOR_DELTA is strictly less expressive than CHUNK_WRITE
(deltas can only modify existing bytes, not overwrite arbitrarily),
so it introduces no new authorization capability.  A malicious
authorized writer can already corrupt the chunk via CHUNK_WRITE; the
delta-write path is not a new attack surface for that principal.

The novel threat model to consider is *partial-application
laundering*: a malicious writer applies deltas to some but not all
projections, hoping to induce repair to lock in incorrect bytes.
This threat is closed by the repair-path rule in {{sec-repair}}:
repair MUST NOT reconstruct from the majority while an epoch is
open.  The epoch-open state on any surviving projection is the
signal that partial application may have occurred; repair either
waits for closure or explicitly rolls back.

## Transport Security

CHUNK_XOR_DELTA MUST be issued over a transport that provides
integrity protection at the RPC layer (for example RPCSEC_GSS with
krb5i {{RFC5661}}, or an equivalent mechanism).  This requirement
binds regardless of the transport-security posture of any other
flexible file v2 layout data-server operation: a delta write applied
to a projection whose integrity is not authenticated is
indistinguishable at the data server from a forged write and cannot be
reconciled by any downstream mechanism defined in this document.  The
wire XDR carries no confidentiality of its own and its integrity is
not self-authenticating.

## Denial of Service via Open Epochs

Per {{sec-epochs}} the data server bounds the delta log for each open
epoch, but the aggregate number of concurrently open epochs held by a
single client across chunks and across files is not bounded by
protocol.  A misbehaving or wedged writer that opens many epochs
without finalizing can:

- Accumulate per-epoch log state on each affected data server, consuming
  bounded but non-trivial data server memory
- Stall repair on every affected chunk, since {{sec-repair}} requires
  the repair coordinator to wait for open epochs to close before
  reconstructing

A data server SHOULD impose an implementation-defined aggregate cap on
the number of concurrent open epochs per client stateid.  On hitting
the cap, further EPOCH_OPEN attempts MUST be rejected with
NFS4ERR_DELAY {{RFC5661}} (retriable at the RPC layer without
protocol-level state change; more idiomatic than NFS4ERR_RESOURCE for
pNFS layout-scoped exhaustion, which is not session-owning). A client
that receives NFS4ERR_DELAY on EPOCH_OPEN MUST NOT retry the same
EPOCH_OPEN immediately; it SHOULD first close (via CHUNK_FINALIZE or
CHUNK_ROLLBACK) at least one of its currently open epochs, then MAY
retry.  Retrying without closing an epoch is protocol-legal but will
typically fail again with NFS4ERR_DELAY.

The metadata server's remedy against an unresponsive-writer scenario
is layout recall or stateid revocation ({{sec-revocation}}), which
forces data server rollback of all epochs bound to the revoked
stateid.

## Retransmit Mismatch

{{sec-concurrency}} specifies that a data server receiving a duplicate
`cxde_seq` in the same epoch MUST NOT re-apply the delta and MUST
return the same result as the original response.  If the retransmit
carries the same (cxda_guard, cxda_owner, cxde_seq) but differs from
the original in `cxde_bin_offset` or `cxde_delta` bytes, this
indicates either a client bug or a wire-level tamper past the
integrity protection in force.  The data server MUST reject such a
mismatch with NFS4ERR_INVAL and SHOULD log the event for
administrative review.

## Log-Capacity Disclosure

The response fields `cxdr_log_bytes_used` and
`cxdr_log_bytes_available` disclose the data server's per-chunk
delta-log occupancy to the client.  This disclosure is intentional --
the client needs the signal to decide whether to close-and-reopen an
epoch or fall back to CHUNK_WRITE.  The information is limited to
principals already authorized to issue CHUNK_XOR_DELTA against the
chunk in question; no additional confidentiality boundary is crossed.

## Optional Ownership Restriction

Deployments that wish to further narrow the writer set for a file MAY
set the layout flag FFV2_FLAGS_DELTA_OWNER_ONLY (defined here,
registered against `ffv2_layout_flags4`).  When set on a layout, data
servers MUST reject CHUNK_XOR_DELTA whose `cxda_owner` does not match
the `chunk_owner4` of the current COMMITTED generation of the target
chunk.  This restricts delta epochs to the writer who last wrote the
chunk, which is the natural semantics for the HPC checkpoint workload
(each rank owns its stride).

This flag is optional; deployments that do not need per-writer
narrowing (single-writer files, files with a well-known writer set)
MAY leave it clear.

## Cryptographic-Checksum Deployments

Deployments that select a cryptographic checksum algorithm
(SHA-family, BLAKE-family) for `ffv2m_checksum_algorithm` cannot use
CHUNK_XOR_DELTA (see {{sec-scope}}).  Such deployments have chosen
adversarial-resistance for chunk envelopes at the cost of some
performance optimizations; this is a coherent, spec-compliant
posture and does not require workaround.  The extension defined here
does not weaken cryptographic-checksum deployments in any way, as
the capability conjunction structurally excludes them.

# IANA Considerations {#sec-iana}

Following the pattern established by the flexible file v2 layout family
({{I-D.haynes-nfsv4-flexfiles-v2}}) that operation numbers
in the NFSv4.2 opnum space are assigned by publication of the
specifying document, operation number 100 is assigned to
CHUNK_XOR_DELTA by publication of this document.  No IANA action
is requested for the operation number.  The current family-wide
NFSv4.2 opcode allocation, of which 100 is the next available
value, is (78-95 being defined in
{{I-D.haynes-nfsv4-flexfiles-v2}}):

- 78-88: CHUNK_COMMIT through CHUNK_WRITE_REPAIR
- 89-91: TRUST_STATEID, REVOKE_STATEID, BULK_REVOKE_STATEID
- 92-95: CHUNK_ESCROW_INSTALL through CHUNK_ESCROW_TAKEOVER
- 96-99: PROXY_REGISTRATION through PROXY_CANCEL (proxy-server
  document, `draft-haynes-nfsv4-flexfiles-v2-proxy-server`)
- 100:   CHUNK_XOR_DELTA (this document)

Opcode values MUST NOT overlap across family documents; a future
extension MUST take the next available value at or above 101, and
MUST NOT re-use any value the family has already allocated.

This document requests the following IANA actions:

## Encoding Registry: XOR-Delta-Capable Column {#sec-iana-encoding-flag}

Add a new column named "XOR-Delta-Capable" (boolean) to the
"Flexible File Version 2 Layout Type Erasure Coding Type Registry"
defined in {{I-D.haynes-nfsv4-flexfiles-v2}}.
The column is called EC_ENC_FLAGS_XOR_DELTA_CAPABLE in prose
references.

Set for encodings whose parity is expressible as an XOR combination
of source bytes AND whose data-shard bytes are directly readable
from a single projection (systematic property).  Clear otherwise.
Initial assignments:

- FFV2_ENCODING_MIRRORED: SET (identity encoding; delta at
  `(chunk_offset, bin_offset)` applied to every mirror at the same
  offset converges to the correct state)
- FFV2_ENCODING_MOJETTE_SYSTEMATIC: SET
- FFV2_ENCODING_MOJETTE_NON_SYSTEMATIC: CLEAR (XOR-linear but
  non-systematic; recovering D_old requires k projection reads and
  an inverse transform, defeating the delta-write purpose.
  Support for non-systematic XOR-linear encodings is deferred to
  a follow-up specification.)
- FFV2_ENCODING_XOR_PARITY: SET
- FFV2_ENCODING_LINUX_MD_RAID: CLEAR (Q shard is GF-multiplicative;
  P shard alone is not sufficient to support delta writes on the
  full parity set)
- FFV2_ENCODING_RS_VANDERMONDE: CLEAR
- FFV2_ENCODING_PASSTHROUGH: not applicable -- PASSTHROUGH layouts
  do not carry the chunk envelope this document operates on;
  CHUNK_XOR_DELTA is undefined for PASSTHROUGH layouts and a data
  server MUST return NFS4ERR_NOTSUPP if received against one.  The
  flag value is nominally CLEAR but the operation itself is
  structurally inapplicable.

## Checksum Registry: XOR-Affine Column {#sec-iana-checksum-flag}

Add a new column named "XOR-Affine" (boolean) to the
"Flexible File Version 2 Layout Type Checksum Algorithm Registry"
defined in {{I-D.haynes-nfsv4-flexfiles-v2}}.  The column is
called CHECKSUM_FLAGS_XOR_AFFINE in prose references.

Set for algorithms that satisfy the XOR-affine identity defined
in {{sec-terminology}}; clear otherwise.  Initial assignments:

- CHECKSUM_ALG_NONE: SET.  The value is SET, not "not
  applicable", so that the eligibility boolean below is a
  well-defined AND with no per-algorithm exceptions.  The
  semantic justification is that XOR-affinity is a property of
  the checksum function; there is no checksum function to fail
  the property when CHECKSUM_ALG_NONE is in use, so the property
  holds vacuously.  The data-server-side checksum-recompute path in
  {{sec-checksum}} is a no-op when CHECKSUM_ALG_NONE is
  configured.
- CHECKSUM_ALG_CRC32: SET
- CHECKSUM_ALG_CRC32C: SET
- CHECKSUM_ALG_FLETCHER4: CLEAR (modular sum; not XOR-linear)
- CHECKSUM_ALG_SHA256: CLEAR (cryptographic hash)
- CHECKSUM_ALG_SHA512: CLEAR (cryptographic hash)
- CHECKSUM_ALG_BLAKE3: CLEAR (cryptographic hash)

A configuration is delta-write-eligible iff both the encoding's
`EC_ENC_FLAGS_XOR_DELTA_CAPABLE` and the checksum's
`CHECKSUM_FLAGS_XOR_AFFINE` are SET.  This is a plain boolean
conjunction with no per-algorithm exceptions; a registry
consumer implementing the conjunction directly derives the
correct answer for every entry, including CHECKSUM_ALG_NONE
(SET AND SET = SET).

## New Error Codes

Following the pattern established by {{I-D.haynes-nfsv4-flexfiles-v2}}
that nfsstat4 codes scoped to the flexible file v2 layout protocol
family are assigned by publication of the specifying document (no IANA
nfsstat4 registry exists), this document assigns:

- NFS4ERR_DELTA_INCOMPLETE = 10110: returned by CHUNK_FINALIZE when
  the recorded delta-log sequence numbers do not form a contiguous
  range from 1 to N.
- NFS4ERR_DELTA_LOG_FULL = 10111: returned by CHUNK_XOR_DELTA when
  the per-chunk delta log would overflow.  The client SHOULD close
  the current epoch and open a new one.

No IANA action is requested for these codes.  The values 10110 and
10111 are chosen to sit above the base specification's cluster (10100 =
NFS4ERR_CHUNK_GUARDED and neighbors) with a gap for future
CHUNK operation codes.

## New Layout Flag

Following the same pattern (no IANA registry for flexible file v2 layout
flags), this document assigns:

- FFV2_FLAGS_DELTA_OWNER_ONLY = 0x00000100: restricts
  CHUNK_XOR_DELTA epochs to the current chunk_owner4 of the target
  chunk.

The bit value 0x00000100 is chosen to sit clear of
FFV2_FLAGS_ONLY_ONE_WRITER (0x00000010, base specification) and the
low-order bits inherited from ff_flags4.  No IANA action is
requested.

--- back

# Worked Example: HPC Checkpoint at 1000 Ranks {#sec-example-hpc}

This appendix walks through the wire-traffic and per-server work
implications of CHUNK_XOR_DELTA versus the base CHUNK_WRITE path for
a canonical HPC checkpoint workload.

## Scenario

- A single file, 1 TB in size, protected by
  FFV2_ENCODING_MOJETTE_SYSTEMATIC at k=8 m=4 with 256 KiB shards
  (2 MiB stripes; 500 000 stripes across the file)
- 1 000 MPI ranks, each responsible for a 1 GiB region of the file
- Checkpoint interval: each rank writes 1 MiB of updated state
  every 10 seconds, distributed across its 1 GiB region as 256
  independent 4 KiB writes
- Ranks are block-aligned per the HPC guidance in
  {{I-D.haynes-nfsv4-flexfiles-v2}}; no two ranks write
  into the same 2 MiB stripe within a single checkpoint interval
- CHECKSUM_ALG_CRC32C checksums (XOR-affine)
- Twelve data servers, one per projection

Per-rank per-interval work: 256 writes of 4 KiB each.  Each 4 KiB
write lives inside a distinct stripe; each stripe's 12 projections
(8 data + 4 parity in FFV2_ENCODING_MOJETTE_SYSTEMATIC terms) live
on the 12 data servers.

## Path A: Base CHUNK_WRITE

In FFV2_ENCODING_MOJETTE_SYSTEMATIC at k=8 m=4, each of the 8 data
projections stores a distinct data shard and each of the 4 parity
projections is an XOR combination of the data shards.  A 4 KiB edit
inside data shard i affects data server i (whose 256 KiB chunk is
rewritten with the mutated shard) plus all 4 parity data servers (each
storing a fresh XOR of the 8 data shards).  Five data servers receive
CHUNK_WRITE; the seven unaffected data servers are untouched.

For each 4 KiB write the client MUST:

1. Read the affected data shard from data server i (256 KiB in, if not
   cached)
2. Compute the new data shard (256 KiB output)
3. Re-encode all 4 parity projections (256 KiB each x 4 = 1 MiB
   output)
4. Transmit the new data shard + 4 new parity projections =
   5 x 256 KiB = 1.25 MiB out, across 5 CHUNK_WRITE requests
5. Await 5 CHUNK_WRITE responses

Per-rank per-write wire cost (Path A, warm-cache case where the
client already has D_old):

- Data server i CHUNK_WRITE payload: 256 KiB (full new chunk)
- 4 x parity data server CHUNK_WRITE payload: 4 x 256 KiB = 1 MiB
- Total wire out per write: 1.25 MiB
- Per-rank per-interval: 256 writes x 1.25 MiB = 320 MiB
- Aggregate across 1 000 ranks per 10-second interval: 312.5 GiB

Per-write client compute: full stripe re-encode (Mojette forward
transform over 2 MiB).  Empirical measurements on commodity
hardware place the Mojette encoder throughput for
FFV2_ENCODING_MOJETTE_SYSTEMATIC at k=4 m=2 in the low-to-mid
gigabytes-per-second range; taking a mid-range figure of 6 GB/s,
a 2 MiB re-encode costs on the order of 330 microseconds.  Per
rank per interval: 256 x 330 microseconds is approximately 85
milliseconds of pure Mojette compute, before RPC and network
overheads.

## Path B: CHUNK_XOR_DELTA

For each 4 KiB write the client:

1. Reads the current 4 KiB of the affected data shard (from data
   data server i) if not cached -- 4 KiB in
2. XORs old with new to produce a 4 KiB delta
3. Sends CHUNK_XOR_DELTA(EPOCH_OPEN + entry) to the 4 parity data
   servers with the 4 KiB delta payload each
4. Also issues CHUNK_XOR_DELTA against the data server i for its own
   byte-range change.  (The base CHUNK_WRITE path in
   {{I-D.haynes-nfsv4-flexfiles-v2}} is a
   whole-chunk-generation producer and does not have a small-write
   fast path; for delta-eligible encodings this document's
   CHUNK_XOR_DELTA is what the client uses to update the data
   shard alongside the parity shards.)
5. At end of checkpoint interval, issues CHUNK_FINALIZE +
   CHUNK_COMMIT on every affected chunk

Per-rank per-write wire cost (Path B):

- Data server i CHUNK_XOR_DELTA payload:
  ~4 KiB + small envelope
- 4 x parity data server CHUNK_XOR_DELTA payload: 4 x 4 KiB = 16 KiB
- Total wire out per write: ~20 KiB
- Per-rank per-interval: 256 writes x 20 KiB = 5 MiB
- Aggregate across 1 000 ranks per 10-second interval: ~5 GiB

Per-write client compute: one XOR of 4 KiB (nanoseconds), plus
per-parity-data-server RPC setup.  No Mojette re-encode.

## Cost Comparison

Path A = CHUNK_WRITE (from {{I-D.haynes-nfsv4-flexfiles-v2}});
Path B = CHUNK_XOR_DELTA (this document).

| Metric                        |    Path A |    Path B |  Ratio |
|-------------------------------|----------:|----------:|-------:|
| Wire out per write            |   1.25MiB |    ~20KiB |    64x |
| Wire out per rank per interval|    320MiB |     5 MiB |    64x |
| Wire aggregate per 1000 ranks | 312.5 GiB |     5 GiB |    62x |
| Client compute per write      |    ~315us |     ~1 us |   300x |
| DS compute per parity per op  | 256 KiB   |    4 KiB  |    64x |

The dominant effect is aggregate wire traffic against fabric
capacity.  Per-rank per-interval Path A is 320 MiB; on a
dedicated 10 Gbps NIC (~1.19 GiB/s) this is a lower bound of
~0.26 s of wire transmission per rank, well within the 10-second
checkpoint interval considered in isolation.  The problem is not
per-rank wall clock; it is aggregate contention.

Aggregate across 1 000 ranks per interval: Path A is 312.5 GiB. On a
shared fabric where per-data-server ingress and cross-rank bandwidth
compete, this saturates commodity checkpoint fabrics -- a 100 GbE core
(~11.9 GiB/s) needs ~26 s just to drain the write volume, exceeding
the interval and backlogging subsequent checkpoints.  Path B's 5 GiB
per interval per 1 000 ranks (~62x reduction) drains in roughly ~0.4 s
on the same fabric, comfortably fitting the interval with headroom for
reads, callbacks, and interference.  The point of the extension is not
the per-rank wall clock but the aggregate-fabric budget: delta writes
convert an at-the-fabric-limit workload into a
comfortably-under-the-limit workload.

## Assumptions and Caveats

The comparison above assumes:

- All parity projections have the same size as data shards
  (holds for FFV2_ENCODING_MOJETTE_SYSTEMATIC by construction).
  Non-systematic encodings are out of scope for this document
  (see {{sec-scope}}).
- The client can XOR at memory-bandwidth speed (holds on
  commodity hardware for 4 KiB payloads)
- Each RPC has non-trivial fixed overhead; the small-payload path
  is not amortized down to zero.  Real deployments will observe
  wire-cost ratios closer to 30-40 x rather than the 64 x
  raw payload ratio, once RPC framing, TCP/RDMA headers, and
  CHUNK_FINALIZE/CHUNK_COMMIT amortization are accounted for.
- No RPC failures or retries; add ~2 x for pessimistic
  reliability accounting on both paths.

Even under the conservative 30 x figure, the wire and compute
savings comfortably justify Path B for the HPC checkpoint workload
class.  For workloads that make large edits (whole-shard or
whole-stripe replacement), Path A is competitive or cheaper (a
full-chunk overwrite avoids the epoch-open + finalize overhead
Path B pays) -- CHUNK_WRITE remains the correct choice there.

# Acknowledgements

The delta-write technique described here builds on longstanding
practice in erasure-coded storage systems: NetApp WAFL's parity
delta logic, Linux md RAID-5/6's P-shard XOR delta path, and Ceph's
partial-parity-update optimization for RADOS erasure-coded pools.
The specific application to XOR-based Mojette in the pNFS
data-server context arose from informal discussions at IETF 126.
