English | [中文版](./NPS-Node-L1.cn.md)

# NPS-Node-L1 Conformance Suite

**Status**: Draft
**Version**: 0.1
**Date**: 2026-04-24
**Applies-To**: [NPS-Node Profile §3](../NPS-Node-Profile.md) — Level 1 Basic
**Authors**: Ori Lynn / INNO LOTUS PTY LTD

> This document defines the test cases an implementation MUST pass to claim
> NPS-Node Profile L1 compliance. Test cases are language-agnostic; the reference
> implementation uses .NET 10 + xUnit under `impl/dotnet/tests/NPS.Daemon.Conformance.Tests/`.

---

## 1. How to Use This Document

1. Build or install the implementation under test (**IUT**).
2. Start a **peer** — any implementation that already passes L1, or the .NET reference SDK.
3. Run every test case in §3 against the IUT paired with the peer.
4. A test case passes if and only if **all** of its acceptance criteria hold.
5. All 21 cases MUST pass for L1 certification; no partial claim is allowed.
6. Copy [`NPS-NODE-L1-CERTIFIED.md`](./NPS-NODE-L1-CERTIFIED.md) to the IUT repository root, fill it in, and sign the attestation with the IUT's root key.

Self-certification is sufficient at this release. Third-party certification (NPS Cloud CA) is targeted for L3 in 2027 Q1+ and is out of scope here.

---

## 2. Test Environment

| Requirement | Value |
|-------------|-------|
| Network | Loopback only; cases do not require external DNS or inter-host routing |
| Peer | Any L1-passing NPS implementation (reference: `NPS.Core` + `NPS.NDP` + `NPS.NIP` + `NPS.NWP` at NPS v1.0.0-alpha.3 or later) |
| Clock | Wall clock within ±5 s of the peer (ISO 8601 UTC timestamps) |
| File system | Writable directory for the IUT's key store and inbox persistence |
| Wire encoding | Tier-1 JSON MUST be exercised; Tier-2 MsgPack cases are deferred to L2 |

Every case runs in a fresh IUT state: delete the IUT's key store / registry / inbox between cases unless the case is explicitly continuation-oriented.

---

## 3. Test Cases

Each case lists the requirement ID from [NPS-Node Profile §3](../NPS-Node-Profile.md), the fixture, the actions, and the acceptance criteria.

### 3.1 NCP — Wire format

#### TC-N1-NCP-01 — Tier-1 JSON frame round-trip
**Req**: N1-NCP-01
**Fixture**: Peer constructs one valid instance of each L1 frame (HelloFrame, AnchorFrame, IdentFrame, AnnounceFrame, ResolveFrame, ActionFrame, CapsFrame, ErrorFrame).
**Action**: Peer sends each frame to the IUT; IUT echoes the decoded frame back re-encoded.
**Pass**:
- IUT decodes every frame without error.
- Re-encoded output is byte-identical to input after RFC 8785 JSON canonicalization.

#### TC-N1-NCP-02 — Hello + Anchor handshake
**Req**: N1-NCP-02
**Fixture**: IUT listening on loopback; peer idle.
**Action**: Peer opens TCP connection, sends HelloFrame, awaits CapsFrame, then publishes an AnchorFrame.
**Pass**:
- IUT responds to HelloFrame with a CapsFrame declaring its supported encoding tiers.
- IUT ACKs the AnchorFrame and caches it for later frame resolution.
- Round-trip completes within 500 ms on loopback.

#### TC-N1-NCP-03 — Loopback listener default
**Req**: N1-NCP-03
**Fixture**: IUT started with default configuration (no `--listen` override).
**Action**: Peer probes `127.0.0.1:17433`.
**Pass**:
- IUT accepts the connection on the default address.
- IUT does **not** accept connections on a non-loopback address (e.g., `0.0.0.0:17433`) unless the operator has explicitly opted in via configuration.

#### TC-N1-NCP-04 — Tier-2 negotiation hygiene
**Req**: N1-NCP-04
**Fixture**: IUT configured without Tier-2 support.
**Action**: Peer sends a HelloFrame advertising Tier-2 only.
**Pass**:
- IUT responds with a CapsFrame listing `tier1-json` in its supported tiers.
- IUT does **not** silently fall back to an unsupported tier; if no common tier exists, IUT returns an ErrorFrame with `NPS-CLIENT-NEGOTIATION-FAILED` (or the peer-observable equivalent).

### 3.2 NIP — Identity

#### TC-N1-NIP-01 — Root keypair generation and permission
**Req**: N1-NIP-01
**Fixture**: Empty key store directory.
**Action**: Start the IUT; locate the persisted root key file.
**Pass**:
- IUT creates exactly one Ed25519 keypair on first start.
- Private-key file permission is `0600` on POSIX; equivalent owner-only ACL on Windows.
- Subsequent starts reuse the same key (no regeneration).

#### TC-N1-NIP-02 — IdentFrame sign and verify
**Req**: N1-NIP-02
**Fixture**: Peer holds a known-good IdentFrame; IUT has its own root key.
**Action**: (a) Peer sends its IdentFrame as the first frame after handshake. (b) Peer opens a second connection and sends an IdentFrame whose signature has been corrupted by flipping one byte.
**Pass**:
- (a) IUT accepts the valid IdentFrame.
- (b) IUT rejects the tampered IdentFrame with an ErrorFrame mapping to `NPS-AUTH-UNAUTHENTICATED`.
- (a) IUT's outbound IdentFrame (sent to peer) verifies successfully under the IUT's root public key.

#### TC-N1-NIP-03 — NID format
**Req**: N1-NIP-03
**Fixture**: IUT freshly initialized.
**Action**: Read IUT's advertised NID (via AnnounceFrame or CLI).
**Pass**:
- NID matches `urn:nps:node:<authority>:<id>` where `<authority>` is non-empty and `<id>` is non-empty.
- NID is stable across restart (does not change after a clean restart with the same key store).

#### TC-N1-NIP-04 — Sub-NID issuance (optional at L1)
**Req**: N1-NIP-04
**Fixture**: IUT supporting sub-NID issuance.
**Action**: Skipped if the IUT declines the capability. Otherwise, a hosted agent requests a session sub-NID.
**Pass**:
- If attempted: issued sub-NID is signed by the root key, contains a non-zero TTL, and verifies under the peer's trust policy.
- If declined: IUT returns a well-formed `NPS-SERVER-NOT-IMPLEMENTED` and the case is recorded as **N/A** rather than failed.

### 3.3 NDP — Discovery

#### TC-N1-NDP-01 — AnnounceFrame carries activation_mode
**Req**: N1-NDP-01
**Fixture**: IUT hosting at least one agent.
**Action**: Peer subscribes to NDP; captures the next AnnounceFrame emitted by the IUT.
**Pass**:
- `activation_mode` field is present.
- Value is exactly `"ephemeral"` (since L1 supports only `ephemeral`).

#### TC-N1-NDP-02 — AnnounceFrame signature
**Req**: N1-NDP-02
**Fixture**: Peer has fetched the IUT's IdentFrame public key.
**Action**: Peer captures an AnnounceFrame from the IUT and verifies the signature.
**Pass**:
- Signature verifies under the declared NID's public key.
- A mutated copy of the captured frame (one byte flipped in any non-signature field) MUST fail verification.

#### TC-N1-NDP-03 — ResolveFrame response
**Req**: N1-NDP-03
**Fixture**: IUT has one hosted agent with known NID `A`.
**Action**: Peer sends a ResolveFrame targeting the `nwp://` URL of agent `A`; peer also sends a ResolveFrame for an unknown NID `Z`.
**Pass**:
- Response for `A` contains the agent's physical endpoint and a non-expired TTL.
- Response for `Z` is an ErrorFrame with `NDP-RESOLVE-NOT-FOUND`.

#### TC-N1-NDP-04 — GraphFrame subscription (optional at L1)
**Req**: N1-NDP-04
**Fixture**: IUT's GraphFrame subscription capability.
**Action**: Skipped or attempted as per capability. Full behavior is validated at L2.
**Pass**:
- If attempted: IUT returns a well-formed initial GraphFrame; **N/A** if declined at L1.

### 3.4 NWP — Inbox and delivery

#### TC-N1-NWP-01 — Inbox accepts ActionFrame
**Req**: N1-NWP-01
**Fixture**: IUT hosting agent `A`; no prior inbox state.
**Action**: Peer sends an ActionFrame addressed to `A`.
**Pass**:
- IUT acknowledges acceptance (no immediate response body required at L1 pull model).
- Peer's subsequent NWP pull for agent `A` returns the deposited frame, bit-identical.

#### TC-N1-NWP-02 — Inbox persists across restart
**Req**: N1-NWP-02
**Fixture**: After TC-N1-NWP-01 completes, before the pull step.
**Action**: Stop the IUT (`SIGTERM` on POSIX / equivalent on Windows); wait for graceful shutdown. Restart the IUT with the same data directory. Peer then issues the NWP pull for agent `A`.
**Pass**:
- Pull returns the undelivered ActionFrame from before the restart, bit-identical.

#### TC-N1-NWP-03 — NWP pull serves inbox
**Req**: N1-NWP-03
**Fixture**: Agent `A` has 3 pending frames in its inbox.
**Action**: Peer issues 3 NWP pull requests in sequence.
**Pass**:
- First pull returns the first (oldest) frame.
- Pulls 2 and 3 return frames 2 and 3 in FIFO order.
- A 4th pull returns an empty result within 1 s (no blocking past inbox drain).

#### TC-N1-NWP-04 — 100 QPS baseline
**Req**: N1-NWP-04
**Fixture**: Agent `A` with a pre-filled inbox of 10 000 small (≤ 1 KB) CapsFrame entries.
**Action**: Peer issues 100 pull requests per second for 10 s (sustained 100 QPS).
**Pass**:
- Zero pull errors.
- 99th-percentile request latency < 100 ms on commodity hardware (1 vCPU / 1 GB RAM baseline).
- Inbox drain count equals requests issued (no duplicates, no drops).

#### TC-N1-NWP-05 — Push path (optional at L1)
**Req**: N1-NWP-05
**Fixture**: IUT's push capability.
**Action**: Skipped if the IUT does not claim push support at L1. Full behavior validated at L2.
**Pass**:
- If attempted: a push delivered to `activation_endpoint` is received bit-identical; **N/A** if declined.

### 3.5 Observability

#### TC-N1-OBS-01 — Frame log entry per direction
**Req**: N1-OBS-01
**Fixture**: IUT started with log output to a capturable destination.
**Action**: Peer initiates a single request/response round trip (AnnounceFrame in, AnnounceFrame ACK out).
**Pass**:
- Log contains at least one entry per frame direction.
- No frame direction goes unlogged.

#### TC-N1-OBS-02 — Log entry fields
**Req**: N1-OBS-02
**Fixture**: Capture of any log entry from TC-N1-OBS-01.
**Action**: Parse the entry (structured formats only — JSON, logfmt, or equivalent; prose-only logs fail this case).
**Pass**:
- Entry contains every field: ISO 8601 UTC timestamp, direction (`in` / `out`), frame type, source NID, destination NID, size in bytes.
- Values are type-correct (timestamp parses as RFC 3339; size is a non-negative integer).

#### TC-N1-OBS-03 — Log destination flexibility
**Req**: N1-OBS-03
**Fixture**: IUT configuration exposes log destination choice.
**Action**: Configure the IUT to emit to (a) stdout and (b) a file, in separate runs.
**Pass**:
- Each destination captures the same log entries as TC-N1-OBS-01/02.
- No requirement to emit to a remote log aggregator at L1.

---

## 4. Results Manifest

A conformance run produces a manifest (JSON) summarizing per-case outcomes. The manifest is embedded into [`NPS-NODE-L1-CERTIFIED.md`](./NPS-NODE-L1-CERTIFIED.md):

```json
{
  "profile": "NPS-Node-L1",
  "profile_version": "0.1",
  "iut": {
    "name": "example-daemon",
    "version": "0.1.0",
    "nid": "urn:nps:node:example.com:host-01"
  },
  "peer": {
    "name": "nps-dotnet-reference",
    "version": "1.0.0-alpha.3"
  },
  "run": {
    "date": "2026-04-24T00:00:00Z",
    "environment": "linux-x64 / 1 vCPU / 1 GB"
  },
  "cases": [
    { "id": "TC-N1-NCP-01", "result": "pass" },
    { "id": "TC-N1-NCP-02", "result": "pass" }
    /* ... 19 more ... */
  ],
  "summary": { "pass": 21, "fail": 0, "skip": 0, "na": 0 }
}
```

Certification is granted when **all 21 cases are `pass` or `na`** (cases N1-NIP-04, N1-NDP-04, N1-NWP-05 may be `na` when the IUT declines the optional capability).

---

## 5. Reference Suite Location

| Language | Path | Status |
|----------|------|--------|
| .NET 10 (xUnit) | `impl/dotnet/tests/NPS.Daemon.Conformance.Tests/` | Planned; tracked alongside NPS Daemon MVP |
| Python | `impl/python/tests/conformance/node_l1/` | TODO (Phase 2) |
| TypeScript | `impl/typescript/tests/conformance/node-l1/` | TODO (Phase 2) |

The reference suite's test names MUST align with the `TC-N1-*` IDs above so a test-run report maps 1:1 onto the §4 manifest.

---

## 6. Change Log

| Version | Date | Changes |
|---------|------|---------|
| 0.1 | 2026-04-24 | Initial draft: 21 test cases covering NCP / NIP / NDP / NWP / Observability, paired-peer methodology, results manifest schema |

---

*Copyright: LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
