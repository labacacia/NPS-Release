English

# NPS-CR-0008: Tier-3 BinaryVector v1 Encoding

**Status**: Proposed  
**Target**: v1.0-alpha.15 or later  
**Date**: 2026-06-27  
**Authors**: Ori Lynn / INNO LOTUS PTY LTD  
**Touches**: NPS-1 NCP, NPS-0 Overview, SDK core codecs, NCP conformance vectors

---

## 1. Summary

Activate the original Tier-3 encoding slot (`Flags.T1T0 = 0b10`) as
**BinaryVector v1**. Tier-3 is a specialized AI-native payload encoding for
vector-heavy frames: metadata is MessagePack and dense vector values are stored
as raw little-endian float32 segments.

This CR deliberately does not turn Tier-3 into Protobuf, a generic opaque
binary blob tier, or a replacement for NWP semantics.

## 2. Motivation

NPS carried the "Tier-3 MatrixTensor" roadmap item from the earliest release
tags, but the implementation work drifted toward simply marking `0b10`
reserved. That left NWP vector search using Tier-1/Tier-2 encodings where each
float is carried as a structured value.

BinaryVector v1 restores the original intent while keeping the first
implementation narrow:

- Tier-1 JSON remains the debug and compatibility tier.
- Tier-2 MessagePack remains the production default for general structured
  frames.
- Tier-3 BinaryVector v1 is optional and negotiated only for vector-heavy
  payloads.

## 3. Specification Changes

NCP v0.9 activates:

- `Flags.T1T0 = 0b10` -> Tier-3 BinaryVector v1.
- Negotiation token: `binary_vector.v1`.
- `Flags.T1T0 = 0b11` remains reserved and invalid.
- Senders MUST NOT emit Tier-3 unless `binary_vector.v1` has been negotiated.
- Receivers that do not support or did not negotiate Tier-3 MUST reject it with
  `NCP-ENCODING-UNSUPPORTED`.

BinaryVector v1 payload layout:

| Offset | Size | Field | Encoding |
|--------|------|-------|----------|
| 0 | 4 | Magic | ASCII `NPBV` |
| 4 | 1 | Version | `0x01` |
| 5 | 1 | Flags | `0x00`; non-zero MUST be rejected |
| 6 | 2 | `vector_count` | uint16, big-endian |
| 8 | 4 | `metadata_len` | uint32, big-endian |
| 12 | 4 | Reserved | MUST be zero |
| 16 | `metadata_len` | Metadata | MessagePack map using Tier-2 field names |
| ... | variable | Vector segments | repeated `dim:uint32_be` + `dim` float32 little-endian values |

Metadata vector fields moved to binary segments are replaced by:

```json
{
  "$nps_binary_vector": 0,
  "dtype": "float32",
  "dim": 1536
}
```

The v0.9 standard binding is NWP `QueryFrame.vector_search.vector`.

## 4. SDK Changes

Required in maintained SDKs:

- Add `EncodingTier.BinaryVector` / equivalent enum value mapped to `0b10`.
- Add a Tier-3 codec that round-trips regular frame metadata and
  `QueryFrame.vector_search.vector`.
- Add negotiation support for `binary_vector.v1`.
- Preserve Tier-1/Tier-2 behavior and reject unnegotiated Tier-3 frames.

.NET reference status:

- `NPS.Core` includes `Tier3BinaryVectorCodec`.
- `NpsFrameCodec` dispatches Tier-3 frames.
- Native-mode handshake negotiates `binary_vector.v1`.
- NWP `QueryFrame.vector_search.vector` round-trips through raw float32
  segments.

Standalone SDK payload-codec status:

- Python: `EncodingTier.BINARY_VECTOR`, Tier-3 payload codec, and payload
  conformance fixture coverage.
- TypeScript: OOP SDK codec plus functional NCP codec support
  `EncodingTier.BINARY_VECTOR` / `EncodingTier.BinaryVector`.
- Go: core header layout migrated to current NCP flags and
  `EncodingTierBinaryVector` payload codec added.
- Rust: core header layout migrated to current NCP flags and
  `EncodingTier::BinaryVector` payload codec added.
- Java: core header layout migrated to current NCP flags and
  `EncodingTier.BINARY_VECTOR` payload codec added.

## 5. Conformance Changes

Updated:

- `ncp.header.006` is now a positive `0b10` BinaryVector v1 header vector.
- `ncp.header.007` is the negative reserved `0b11` header vector.

Added:

- BinaryVector payload fixture for NWP `QueryFrame.vector_search.vector`.
- Malformed payload fixtures for bad magic, reserved flags, marker index
  mismatch, dimension mismatch, unsupported dtype, and truncated vector segment.
- Encoding-policy fixtures proving receivers reject unnegotiated Tier-3 and
  forbid arbitrary per-frame Tier-3 switching outside the standard binding.

## 6. Migration Impact

Tier-3 is opt-in. Existing peers that only advertise `json` and `msgpack` keep
their current behavior. A peer that receives `0b10` without negotiating
`binary_vector.v1` must reject it instead of silently decoding or downgrading.

## 7. Out of Scope

- Protobuf as an NCP encoding tier.
- A generic opaque binary blob tier.
- MatrixTensor layout, float16, quantized int8, sparse vectors, or multi-vector
  schema bindings.
- Replacing NWP vector search semantics.

## 8. Acceptance Criteria

- [x] NCP / NPS-0 / frame-model docs define active BinaryVector v1.
- [x] NCP conformance changes `0b10` to positive and keeps `0b11` negative.
- [x] .NET reference implementation encodes/decodes Tier-3 frames.
- [x] .NET native handshake negotiates `binary_vector.v1`.
- [x] Payload-level conformance fixtures are added.
- [x] Maintained standalone SDKs implement Tier-3 header/payload codec parity.
- [x] Negotiation/policy fixtures prove peers reject unnegotiated Tier-3.
- [x] Malformed payload fixtures cover bad magic, reserved flags, bad marker,
      unsupported dtype, and truncated vector segment.

## 9. Proposed Changelog Entry

Activate NCP Tier-3 BinaryVector v1 (`Flags.T1T0 = 0b10`) with negotiation
token `binary_vector.v1`, `NPBV` payload layout, MessagePack metadata markers,
and little-endian float32 vector segments for NWP `QueryFrame.vector_search.vector`.
