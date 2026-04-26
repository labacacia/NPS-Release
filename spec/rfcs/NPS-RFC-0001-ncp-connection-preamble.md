English | [中文版](./NPS-RFC-0001-ncp-connection-preamble.cn.md)

---
**RFC Number**: NPS-RFC-0001
**Title**: Add NCP connection preamble for native-mode traffic identification
**Status**: Accepted (Phase 1 — spec + .NET reference implementation landed)
**Author(s)**: Ori Lynn <iamzerolin@gmail.com> (LabAcacia)
**Shepherd**: Ori Lynn (pre-1.0 fast-track per `spec/cr/README.md`)
**Created**: 2026-04-21
**Last-Updated**: 2026-04-25
**Accepted**: 2026-04-25 (pre-1.0 fast-track; see `spec/cr/README.md`)
**Activated**: _(set when first reference SDK ships, target v1.0-alpha.3)_
**Supersedes**: _none_
**Superseded-By**: _none_
**Affected Specs**: NPS-1 NCP, spec/error-codes.md, spec/status-codes.md
**Affected SDKs**: .NET, Python, TypeScript, Java, Rust, Go
---

# NPS-RFC-0001: Add NCP connection preamble for native-mode traffic identification

## 1. Summary

Define an 8-byte constant preamble `b"NPS/1.0\n"` that every NCP **native
mode** client MUST send once, immediately after the transport (TCP / QUIC)
handshake and before its first `HelloFrame`. Servers that see any other
byte sequence in those 8 bytes close the connection without emitting an
`ErrorFrame`. HTTP mode is **not** affected. Adds one error code
(`NCP-PREAMBLE-INVALID`) and one status code (`NPS-PROTO-PREAMBLE-INVALID`).

## 2. Motivation

NCP native mode currently relies on `HelloFrame (0x06)` as the
first-byte-after-handshake signal. This works for well-behaved peers but
has two operational weaknesses:

1. **Misrouted traffic is hard to reject cheaply.** If a non-NPS client
   (HTTP/1.x probe, Redis client, port scanner, misconfigured reverse
   proxy) hits the native-mode port, the server parses the first byte as
   a frame type, reads a bogus length field, then eventually fails on a
   downstream parse. A constant preamble lets the server reject in the
   first `read(8)` with zero frame-parser exposure.

2. **Major-version bumps have no wire-level gate.** An NPS 2.0 client
   connecting to a 1.x server gets a `CapsFrame` negotiation failure
   mid-handshake instead of a clean "wrong protocol version" rejection
   before any frame parsing happens. Embedding the major version in the
   preamble gives us a cheap, explicit compatibility check at the byte
   level.

This also answers a review comment from 2026-04-20 arguing NCP "needs a
magic code to handle TCP packet-boundary ambiguity". The commenter
conflated two concerns (length-prefix framing vs. traffic identification)
— NCP's length-prefixed header already handles framing (see §5.2), but
connection-level traffic identification is a legitimate separate concern
that this RFC addresses.

## 3. Non-Goals

This RFC does **NOT**:

- Add any per-frame magic bytes. NCP frames remain length-prefixed with
  no per-frame marker.
- Change HTTP mode. HTTP already disambiguates via
  `Content-Type: application/nwp-frame`.
- Define resync semantics after mid-stream corruption. The policy
  remains "close the connection and reconnect" — same as HTTP/2, gRPC,
  TLS record layer.
- Alter `HelloFrame` or `CapsFrame` negotiation semantics.
- Address QUIC `ALPN` string selection. ALPN is an orthogonal discovery
  mechanism; a future RFC may mandate an ALPN token like `nps/1` for
  native mode over QUIC, but that is not this RFC.

## 4. Detailed Design

### 4.1 Wire Format / Frame Changes

**Preamble byte sequence** (sent once per connection, before any frame):

```
Offset  0   1   2   3   4   5   6   7
       ┌───┬───┬───┬───┬───┬───┬───┬───┐
       │ N │ P │ S │ / │ 1 │ . │ 0 │\n │
       └───┴───┴───┴───┴───┴───┴───┴───┘
       0x4E 50  53  2F  31  2E  30  0A
```

8 bytes, ASCII, ends with `0x0A` (LF). Hex: `4E 50 53 2F 31 2E 30 0A`.

**Why ASCII + LF:**
- `tcpdump`/Wireshark show it as `NPS/1.0\n` inline — zero ambiguity in
  packet captures.
- The leading `N` (`0x4E`) does not collide with any assigned NCP frame
  type (AnchorFrame `0x01`..ErrorFrame `0xFE` — the `0x40`–`0x4F` range
  is currently used for NWP frames `0x41`–`0x43`, so `0x4E` is
  Reserved; §4.3 formalizes the reservation).
- Trailing `\n` gives line-oriented protocol inspectors a clean stop.
- `HTTP/1.1` starts with `HEAD`/`GET `/`POST` etc. — `NPS/` cannot be
  parsed as a valid HTTP request line by any compliant HTTP parser.
- `HTTP/2` preamble is `PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n` (24 bytes,
  starts with `PRI `) — distinct.

**Handshake flow (native mode):**

```
Client                                        Server
  │                                              │
  │──── TCP/QUIC handshake ───────────────────── │
  │                                              │
  │──── 8 bytes "NPS/1.0\n"  (preamble) ──────→ │
  │                                              │  validate
  │                                              │  (close on mismatch,
  │                                              │   no ErrorFrame)
  │──── HelloFrame (0x06) ────────────────────→ │
  │                                              │
  │ ←─── CapsFrame (0x04) ────────────────────── │
  │                                              │
  │──── application frames ←→ ───────────────── │
```

The preamble is **not** a frame — it has no header, no Flags, no length
field. It is raw constant bytes.

**Server validation rules:**

1. On accepting a native-mode connection, server reads exactly 8 bytes.
2. If the 8 bytes match `b"NPS/1.0\n"`, proceed to frame parsing.
3. If they do not match:
   - The server MUST close the connection within 500 ms.
   - The server MUST NOT send an `ErrorFrame` (the peer is not known to
     speak NCP; an `ErrorFrame` write would leak framing details to
     scanners).
   - The server MAY log the first 32 bytes of received traffic for
     operational diagnosis.
4. If fewer than 8 bytes arrive within 10 s, server treats it as a
   timeout and closes silently.

**Client behavior:**

- Client MUST write the 8 preamble bytes in a single buffer (do not
  interleave with application logic; do not split across multiple
  writes unless forced by transport-level MTU, which for 8 bytes is
  never the case in practice).
- Client MAY begin writing `HelloFrame` back-to-back with the preamble
  (no round-trip wait), since server validates byte-by-byte against a
  constant.

**Future major-version semantics:**

- `b"NPS/2.0\n"` etc. are reserved for future major versions.
- A v1.x server that reads `NPS/2.0\n` MUST close the connection
  (unknown major), MAY write a one-time 32-byte diagnostic line
  `NPS-PREAMBLE-UNSUPPORTED-VERSION\n` before closing. This is
  allowed because the peer has self-identified as an NPS speaker.
- Minor version (the `0` in `1.0`) is reserved; v1 clients MUST send
  `0` and v1 servers MUST accept `0` only. A future RFC may define
  minor-version negotiation.

### 4.2 Manifest / NWM Changes

None. The preamble is transport-level, below any NWM surface.

### 4.3 Error Codes

**New error code** in `spec/error-codes.md`:

| Error Code | NPS Status | Description |
|------------|------------|-------------|
| `NCP-PREAMBLE-INVALID` | `NPS-PROTO-PREAMBLE-INVALID` | Client sent invalid or malformed preamble; connection closed without frame-level response |

**New status code** in `spec/status-codes.md`:

| NPS Status | HTTP Mapping | Description |
|------------|--------------|-------------|
| `NPS-PROTO-PREAMBLE-INVALID` | `400 Bad Request` (not emitted; native-mode only) | Native-mode preamble mismatch |

Note: this status is never transmitted on the wire because the server
closes silently. It exists so that SDK-internal telemetry (logs,
metrics) can classify the close reason consistently.

**Frame-type namespace reservation** in `spec/frame-registry.yaml`:

Add a note that the byte value `0x4E` (ASCII `N`), when seen as the
first byte of a native-mode connection, is interpreted as the start of
the preamble, not as a frame type. Frame type `0x4E` MUST NOT be
assigned to any NCP frame.

### 4.4 State Machines / Flows

**Native-mode connection state (server side):**

```
[LISTEN] ──accept──→ [PREAMBLE-WAIT] ──8 bytes match──→ [FRAMING]
                           │                                 │
                           │──mismatch──→ [CLOSING]          │
                           │──timeout(10s)──→ [CLOSING]     │
                                                             │
                                           [FRAMING] ──────→ [HANDSHAKE]
                                                      HelloFrame
                                                             │
                                           [HANDSHAKE] ────→ [ESTABLISHED]
                                                      CapsFrame sent
```

Timeouts:
- `PREAMBLE-WAIT` → `CLOSING`: **10 seconds** (covers slow links + TLS
  handshake tail; aggressive enough to defend against slowloris).
- `CLOSING`: close within **500 ms** after decision.

### 4.5 Backward Compatibility

- Will old Agents understand the new preamble requirement? **No** — this
  is a breaking change for any native-mode client that predates this
  RFC. HTTP mode clients are unaffected.
- Will old Nodes ignore the preamble safely? **No** — old servers will
  read the preamble as a frame header and fail parsing. The first server
  update MUST precede the first client update, or clients will get
  `NCP-FRAME-UNKNOWN-TYPE` from old servers.
- What does a version-mismatch error look like on the wire? Silent close
  from server, 10-second soft timeout from client, client surfaces
  `NCP-PREAMBLE-INVALID` in its own logs.
- Does `min_agent_version` need to be raised? **Yes.** Bump to the
  first version that implements this RFC — 21-day window applies.

Breaking-change rationale: native mode is still Phase 2+ per
`spec/NPS-Roadmap.md`; no GA shipments depend on native mode yet.
Taking the break now is cheaper than retrofitting after 1.0 GA.

---

## 5. Alternatives Considered

### 5.1 Per-frame magic bytes

Prefix every frame with a 2-byte magic (e.g. `0x4E 0x50`).

- **Cost**: 2 bytes × every frame × every connection. For a chatty
  agent sending 10 k frames/s, that's 20 KB/s of pure overhead per
  connection. Unacceptable for the performance target native mode
  exists to serve.
- **Benefit**: mid-stream resync after corruption.
- **Verdict**: rejected. TCP/QUIC already guarantee in-order delivery;
  in-stream corruption means the connection is untrustworthy —
  reconnecting is cheaper than resync anyway. Same rationale as HTTP/2,
  gRPC, TLS record.

### 5.2 Do nothing (rely on HelloFrame as implicit preamble)

Keep the current design: first frame is `HelloFrame (0x06)`; server
parses it; non-NPS traffic fails on frame parse.

- **Cost**: server exposes its frame parser to arbitrary byte sequences
  from port scanners and misrouted traffic. Not yet a known CVE
  vector, but future parser vulnerabilities (malformed MsgPack,
  oversized length fields) would be reachable by unauthenticated
  remote bytes.
- **Ceiling on deferral**: acceptable through Phase 2 (experimental
  native mode). Becomes a blocker before native mode reaches "stable"
  status; we'd rather land it now than do it as a breaking change
  post-1.0 GA.
- **Verdict**: deferring is viable but strictly worse than adopting.

### 5.3 Binary magic (4 bytes, non-ASCII)

E.g., `0x89 4E 50 53` (PNG-style high-bit + "NPS").

- **Cost**: less human-readable in `tcpdump`; loses the "HTTP parser
  rejects immediately" property.
- **Benefit**: slightly shorter (4 bytes vs 8 bytes). Marginal.
- **Verdict**: rejected. The operator-ergonomics win from ASCII is
  worth 4 extra bytes per connection (not per frame).

### 5.4 TLS ALPN token only

Rely on TLS ALPN (`nps/1`) for protocol identification; no in-band
preamble.

- **Cost**: only works over TLS. Plain-TCP native mode (Phase 2 testing,
  on-host IPC scenarios) would have no identification.
- **Benefit**: zero on-wire overhead.
- **Verdict**: rejected as **sole** mechanism, but a future RFC
  SHOULD mandate `nps/1` ALPN for TLS-over-QUIC/TCP native mode as a
  **complement** to the preamble. Defense in depth.

---

## 6. Drawbacks & Risks

- **Migration cost**: ~1 SDK-week per language to add preamble emit +
  validate paths + tests. 6 SDKs = ~6 SDK-weeks total. Two of the six
  (Go, .NET) have the most native-mode code today.
- **Attack surface**: new parser path in the server, but trivially
  small — 8-byte constant comparison, no allocation, no state machine
  depth. Lower attack surface than the frame parser it gates.
- **Ecosystem fragmentation**: none — no outside-project NCP consumers
  exist yet.
- **Reversibility**: removing the preamble later would be a second
  breaking change. Not zero cost. Estimated at ~1 SDK-week if we
  ever needed to roll back.

---

## 7. Security Considerations

- **New parser paths?** Yes, but bounded: 8-byte `memcmp`-equivalent,
  no dynamic allocation, no length-field parsing, no unicode handling.
  Strictly easier to audit than the existing frame header parser.
- **New cryptographic assumptions?** None.
- **Impact on existing signature / encryption / replay protections?**
  None — preamble is pre-TLS in plain-TCP mode, and pre-NIP in all
  modes; no signed material involved.
- **DoS vectors?**
  - *Slowloris-style preamble starvation*: mitigated by 10-second
    `PREAMBLE-WAIT` timeout.
  - *Connection flood*: unchanged from status quo. Rate-limiting is the
    responsibility of the transport/admission layer, not NCP.
  - *Preamble-based fingerprinting*: preamble is a public constant,
    so scanners can identify NPS servers by connecting and sending the
    preamble. This is by design (same tradeoff HTTP/2 made); security-
    sensitive deployments should use a non-default port and/or ALPN
    over TLS.

---

## 8. Implementation Plan

### 8.1 Phasing

| Phase | Scope | Exit criterion |
|-------|-------|----------------|
| 1 | Spec merged + .NET reference implementation behind `NpsNativeOptions.RequirePreamble` flag (default false) | Unit tests green; cross-version interop test (preamble-aware client ↔ preamble-aware server) |
| 2 | All 6 SDKs implement with flag default false | Cross-SDK interop matrix green; at least one SDK's native-mode sample runs end-to-end with preamble on both sides |
| 3 | Flag default flips to **true** across SDKs; release notes call out the break | No open regressions for 1 release cycle; `min_agent_version` bumped with 21-day deprecation window |
| 4 | Remove the flag — preamble mandatory | Flag removal PR lands in each SDK; docs updated |

### 8.2 SDK Coverage Matrix

| SDK | Owner | Status | Notes |
|-----|-------|--------|-------|
| .NET | Ori Lynn | pending | Primary reference; lands first |
| Python | _TBD_ | pending | |
| TypeScript | _TBD_ | pending | Browser: native mode not applicable; Node.js only |
| Java | _TBD_ | pending | |
| Rust | _TBD_ | pending | |
| Go | _TBD_ | pending | |

### 8.3 Test Plan

New tests that land with this RFC:

1. **Preamble-valid round-trip**: client sends `NPS/1.0\n` + HelloFrame;
   server accepts, returns CapsFrame.
2. **Preamble-invalid rejection**: client sends arbitrary 8 bytes
   (`"GET / HTT"`, all-zero, `NPS/2.0\n`); server closes within 500 ms
   with no frame response.
3. **Preamble-truncated**: client sends 3 bytes and pauses; server
   closes after 10-second timeout.
4. **HTTP mode unaffected**: HTTP-mode requests continue to succeed with
   no preamble expected.
5. **Version-future rejection**: client sends `NPS/2.0\n`; server MAY
   write `NPS-PREAMBLE-UNSUPPORTED-VERSION\n` diagnostic before close
   (test asserts close happens, diagnostic is optional).

Existing test changes:

- Native-mode handshake tests in every SDK gain a preamble-emit step
  (once Phase 3 flips default-on, tests without preamble must be
  updated or deleted).

Cross-SDK interop:

- Phase 2 exit requires: preamble-aware .NET server × preamble-aware
  client in each other SDK → handshake completes.

### 8.4 Benchmarks

No new benchmark needed. Wire-size impact is 8 bytes per connection,
amortized to zero for any connection that exchanges >1 frame. Latency
impact is zero because client can pipeline preamble + HelloFrame in a
single write.

If a future measurement shows the 10-second `PREAMBLE-WAIT` timeout
blocks >0.1% of legitimate connections, reopen this RFC's open
question #2.

---

## 9. Empirical Data

None yet. An experimental branch will be attached before moving to
`Accepted`. Target measurements:

| Metric | Baseline | Proposed | Delta | Method |
|--------|----------|----------|-------|--------|
| Per-connection wire overhead | 0 bytes | 8 bytes | +8 B | trivial |
| Handshake RTT (localhost loopback) | TBD | TBD | target: no regression | .NET `BenchmarkDotNet` on native-mode loopback |
| Server-side cost to reject non-NPS scan | full frame-parse attempt | 8-byte `memcmp` | target: ≥10× cheaper | scan-simulation harness |

---

## 10. Open Questions

- [ ] **OQ-1**: Should the preamble include a capability bitmap byte to
  let servers pick connection-level features (e.g., "client supports
  E2E encryption") before `HelloFrame`? Shepherd decision needed.
  _Default position: no — `HelloFrame` already carries full capability
  declaration; preamble stays minimal._ Target: resolved before
  `Accepted`.
- [ ] **OQ-2**: `PREAMBLE-WAIT` timeout of 10 s — should this be
  configurable? Owner: Ori Lynn. Target: default stays 10 s, SDK MAY
  expose a knob; resolved by adding a note to this RFC before
  `Accepted`.
- [ ] **OQ-3**: Should a native-mode **server** also send a preamble
  (mutual identification)? Currently only client → server. Defer to a
  follow-up RFC if requested.

---

## 11. Future Work

- **Follow-up RFC**: mandate TLS ALPN token `nps/1` for native mode
  over TLS-enabled transports. Complements the in-band preamble (§5.4).
- **Follow-up RFC**: formalize minor-version negotiation semantics
  (reserved byte in preamble currently fixed to `0`).
- **Follow-up RFC (if requested)**: server-emitted preamble for mutual
  identification (OQ-3).

---

## 12. References

- [NPS-1-NCP.md §3 Frame Format](../NPS-1-NCP.md) — length-prefix framing
- [NPS-1-NCP.md §2.6 Connection Handshake](../NPS-1-NCP.md) — HelloFrame / CapsFrame flow
- RFC 7540 §3.5 "HTTP/2 Connection Preface" — prior art
- RFC 8446 §4 "TLS 1.3 Handshake" — ClientHello structure as prior art
- Discussion thread: 2026-04-20 review comment on NCP stream framing
  (internal; not yet on Discussions)

---

## Appendix A. Revision History

| Date | Author | Change |
|------|--------|--------|
| 2026-04-21 | Ori Lynn | Initial draft |
| 2026-04-25 | Ori Lynn | Accepted via pre-1.0 fast-track. Spec changes landed: §2.6.1 in NPS-1-NCP, error code `NCP-PREAMBLE-INVALID`, status code `NPS-PROTO-PREAMBLE-INVALID`, `0x4E` reservation in `frame-registry.yaml`. Phase 1 .NET reference helpers (`NPS.Core.Ncp.NcpPreamble`) landed alongside; Phase 2 (other 5 SDKs) and Phase 3 (default-on flip) deferred per RFC §8.1. |
