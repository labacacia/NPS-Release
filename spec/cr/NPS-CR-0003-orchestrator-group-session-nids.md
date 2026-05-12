<!--
Copyright 2026 INNO LOTUS PTY LTD
Developed under LabAcacia Open Source Initiative
Licensed under the Apache License, Version 2.0
-->

# NPS Change Request: Orchestrator Group NIDs and Short-Lived Session NIDs

**CR ID**: NPS-CR-0003
**Target version**: v1.0-alpha.6
**Status**: Accepted
**Accepted**: 2026-05-11
**Type**: Backward-compatible extension (new identifier conventions, new IdentFrame field, new CA endpoints, new revocation reason)
**Author**: Ori, LabAcacia
**Affected components**: NIP spec (NPS-3), NIP error codes, .NET SDK (`NPS.NIP`), `nip-ca-server`, conformance tests, public docs

---

## 1. Summary

NPS today issues **one** NID per identity. An orchestrator that runs many short-lived task executions has only two unsatisfactory options:

- **(a)** Reuse a single long-lived NID across every execution. The blast radius of a key leak is the entire NID's validity window (default 30 days for agent NIDs); incidents cannot be scoped to one execution.
- **(b)** Re-register a new top-level NID every execution. Round-trips an Operator credential through the CA each time, leaves no link between an execution and the orchestrator that produced it, and pollutes the CA's NID space.

This CR adds a first-class **group / session** NID hierarchy:

- A **group NID** represents the orchestrator identity. It is registered once by an Operator, holds a longer validity window (default 365 days), and is the trust anchor for the orchestrator's session NIDs.
- A **session NID** represents one short-lived work execution. It is issued by the CA on demand, has a short validity window (default 1 hour, max 24 hours), and carries verifiable signed lineage metadata that chains it back to its group.
- Issuing a session requires authorization either by the group's own private key (Ed25519 JWS over the request) or by an Operator credential (escape hatch).
- Revoking a group cascades to all live sessions and blocks future session issuance for that group.

The full trust chain is:

```
human owner  →  Operator key (NIP §2.1)  →  orchestrator group NID  →  short-lived session NID
```

Existing single-NID flows (`POST /v1/agents/register`, etc.) are unchanged and remain the default. Group / session is opt-in.

## 2. Motivation

The orchestrator-with-many-short-tasks pattern is dominant in production AI Agent deployments — a long-running planner spawns dozens to thousands of isolated tool-call or subtask executions per hour. Without a group / session split:

- **Blast radius**: a leaked key invalidates 30 days of agent activity instead of (at most) 1 hour of one session.
- **Audit gap**: a Node receiving a request has no protocol-level way to tell which orchestrator produced this execution. Reputation entries (NIP §5.1.2) and incident response cannot fan-in by orchestrator.
- **Operator-credential bottleneck**: every new execution needs an Operator-authed `register` call. Operator API keys grow into hot, broadly-shared secrets — exactly what NIP §10 was designed to avoid.
- **NID-space pollution**: tens of thousands of unrelated agent NIDs accumulate at a CA where really there is one orchestrator with many ephemeral executions.

The pattern is well-precedented:
- **AWS STS** — long-lived IAM users issue short-lived role-session credentials.
- **OAuth 2.0** — long-lived clients exchange refresh tokens for short-lived access tokens.
- **OpenID Connect** — issuers mint short-lived ID tokens for end-user sessions.

NIP needs the equivalent at the NID layer.

## 3. Specification changes

### 3.1 NID identifier conventions (NPS-3 §3)

Add to `spec/NPS-3-NIP.md` §3 NID Format, after the examples:

> **Reserved identifier prefixes** (NPS-CR-0003)
>
> The following identifier prefixes are reserved on NIDs of `entity-type = agent` and signal a structural role within the orchestrator / session lineage model. They MUST follow the existing identifier ABNF (`1*(ALPHA / DIGIT / "-" / "_" / ".")`).
>
> | Prefix | Role | Example |
> |---|---|---|
> | `group-` | Orchestrator group NID. The trust anchor for a fleet of session NIDs issued by the CA on the group's behalf. | `urn:nps:agent:ca.example.com:group-7f3c9e1a-b2d8-4c6f-9a01-...` |
> | `session-` | Short-lived session NID issued under a group. The portion after `session-` MUST be a unique identifier of the form `{unix-timestamp}-{random}` where `{random}` is at least 8 hex characters. | `urn:nps:agent:ca.example.com:session-1714672800-f3a92c0b` |
>
> Identifiers without these prefixes continue to behave as ordinary agent NIDs. Receivers MUST NOT reject a NID solely on its prefix; the prefix is informational and the authoritative role is carried in `IdentFrame.lineage.role` (§5.1.3).

`entity-type` is **not** extended — group and session NIDs remain `agent` so that pre-CR-0003 verifiers parse and route them as ordinary agents (safe degradation: they cannot exploit lineage but cannot be tricked by it either). The protocol-level `nop:orchestrate` capability (NPS-5) continues to express the *role* of being an orchestrator and remains independent of this CR.

### 3.2 IdentFrame.lineage (NPS-3 §5.1)

Add a new optional top-level field to `IdentFrame`:

```json
"lineage": {
  "role":          "group" | "session",
  "parent_nid":    "urn:nps:agent:...",
  "group_nid":     "urn:nps:agent:...",
  "session_id":    "session-1714672800-f3a92c0b",
  "purpose":       "data-extraction-job-42",
  "owner_user_id": "user-123",
  "owner_key_id":  "kid-456"
}
```

| Sub-field | Type | Required | Description |
|---|---|---|---|
| `role` | enum | required when `lineage` is present | `"group"` for an orchestrator group NID; `"session"` for a short-lived session NID. |
| `parent_nid` | string (NID) | required when `role = "session"` | The immediate parent NID. For a 1-level chain (group → session), equals `group_nid`. |
| `group_nid` | string (NID) | required when `role = "session"` | The group NID at the root of the chain. Equals `parent_nid` for a 1-level chain. Reserved for future deeper chains. |
| `session_id` | string | required when `role = "session"` | Stable id of this execution; matches the `session-...` portion of the NID identifier. Echoes back to allow indexing without parsing the URN. |
| `purpose` | string | optional | Free-form, human-readable label for what this execution is for (e.g. `"order-classification-job"`). Bounded to 256 UTF-8 bytes. |
| `owner_user_id` | string | optional | Stable identifier of the human owner this group is acting on behalf of (e.g. an internal user UUID). Required when `role = "group"` and the deployment knows the owner. |
| `owner_key_id` | string | optional | `kid` hint identifying the owner-key (Operator key, OIDC `sub`, hardware-token id) that authorized creation of the group. |

**Signature rules**: `lineage` is part of the **signed** canonical JSON (unlike `metadata`, which is excluded). Any modification to lineage invalidates the IdentFrame signature. This is what makes the chain *verifiable* — a Node can check that a session NID's `lineage.parent_nid` was authoritatively asserted by the CA, not bolted on later.

**Backward compatibility**: pre-CR-0003 publishers omit `lineage`; pre-CR-0003 verifiers are strict-canonical (sort + omit `signature` / `metadata` / `cert_format` / `cert_chain`) and therefore drop unknown fields if their DTO does not carry them, in which case signature verification of a CR-0003 frame **will fail**. This matches the phase model used for NPS-RFC-0003 `assurance_level`: new senders aware of CR-0003 produce CR-0003 frames; old verifiers MUST be upgraded to verify them. The default identity mint (regular agent NIDs) does not include `lineage` and remains exactly bit-compatible.

### 3.3 New revocation reason `parent_revoked` (NPS-3 §5.3)

Extend the `RevokeFrame.reason` enum:

| Reason | When |
|---|---|
| `parent_revoked` (new) | Set on a session-NID `RevokeFrame` that the CA emits as part of cascade revocation when the session's group NID was revoked. Distinguishes a session that was forcibly invalidated by ancestry from one revoked on its own merits. |

The five existing reasons (`key_compromise` / `ca_compromise` / `affiliation_changed` / `superseded` / `cessation_of_operation`) are unchanged.

### 3.4 New CA Server API endpoints (NPS-3 §8)

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/v1/orchestrators/groups/register` | Operator API key | Register an orchestrator group; returns the group's IdentFrame with `lineage.role = "group"`. |
| POST | `/v1/orchestrators/groups/{group_nid}/sessions/issue` | Group JWS **or** Operator API key | Issue a short-lived session NID under the group. Returns the session's IdentFrame with `lineage.role = "session"`. |
| POST | `/v1/orchestrators/groups/{group_nid}/revoke` | Operator API key | Revoke the group AND cascade-revoke every live session whose `lineage.group_nid` equals this group. |
| GET | `/v1/orchestrators/groups/{group_nid}/sessions` | Operator API key | List sessions issued under this group (audit). |

Discovery (`/.well-known/nps-ca`) gains a `capabilities` entry `"orchestrator-group"` so clients can detect support.

### 3.5 Group-JWS authorization shape (§8 addendum)

When an orchestrator authenticates with its group key (no Operator credential), the request body MUST be a flattened JWS with these protected-header fields and a JSON payload:

```jsonc
// Protected header (b64url-encoded)
{
  "alg":         "EdDSA",
  "kid":         "<group_nid>",
  "nps-purpose": "session-issue"
}
```

```jsonc
// Payload (b64url-encoded)
{
  "session_pub_key":  "ed25519:...",        // session keypair public half
  "purpose":          "data-extraction",    // optional, ≤256 bytes
  "validity_seconds": 3600,                  // optional; clamped to [60, MaxValidity]
  "scope_json":       {...},                 // optional; MUST NOT exceed group scope
  "iat":              1714672800             // unix seconds, ±MaxClockSkew
}
```

```jsonc
// Outer JWS object (Content-Type: application/jose+json)
{
  "protected": "<b64url(header)>",
  "payload":   "<b64url(payload)>",
  "signature": "<b64url(Ed25519 signature)>"
}
```

The CA MUST:

1. b64url-decode the protected header and reject unless `alg = EdDSA` and `nps-purpose = session-issue`.
2. Resolve `kid` to a stored group NID; reject `NIP-CA-PARENT-NOT-FOUND` if not found, `NIP-CA-PARENT-NOT-GROUP` if the NID exists but `lineage.role ≠ "group"`, `NIP-CA-GROUP-REVOKED` if revoked.
3. Verify the signature using the group's stored public key. Reject `NIP-CA-JWS-INVALID` on failure.
4. Decode payload, reject `NIP-CA-JWS-EXPIRED` if `|now − iat| > MaxClockSkew` (default 5 minutes).
5. Reject `NIP-CA-SESSION-VALIDITY-INVALID` if `validity_seconds` is outside the configured `[1 minute, MaxSessionValidity]` band.
6. If `scope_json` is supplied, verify it is a non-strict subset of the group's scope (no scope expansion per NIP §10.3).

When an Operator API key is presented (`Authorization: Bearer ...`), the body is plain JSON (no JWS wrapper). This path exists for break-glass / tooling and SHOULD be locked down by deployment-side rate limits.

### 3.6 Verification flow chain check (§7)

Insert a step **3a** in `spec/NPS-3-NIP.md` §7 between the existing signature check (3) and OCSP lookup (4):

> **3a. If the IdentFrame carries `lineage.parent_nid`, the verifier MUST OCSP-lookup the parent NID and reject `NIP-CERT-PARENT-REVOKED` if the parent is revoked or expired. The lookup is mandatory regardless of whether the session NID itself is still valid.**

This is the verify-side half of cascade revocation; combined with the CA-side cascade in §3.3 it provides defense-in-depth: a misconfigured CA that fails to record a cascade still gets caught at the verifier, and a CA that did record it still publishes the entries to the CRL for simple consumers.

### 3.7 New error codes (NPS-3 §9 / `spec/error-codes.md`)

| Error Code | NPS Status | Description |
|---|---|---|
| `NIP-CA-GROUP-REVOKED` | `NPS-AUTH-FORBIDDEN` | Cannot issue a session under a group that has been revoked. |
| `NIP-CA-PARENT-NOT-FOUND` | `NPS-CLIENT-NOT-FOUND` | The `parent_nid` / group NID referenced by a session-issue request does not exist. |
| `NIP-CA-PARENT-NOT-GROUP` | `NPS-CLIENT-BAD-PARAM` | The referenced parent NID exists but is not registered as `lineage.role = "group"`. |
| `NIP-CA-SESSION-VALIDITY-INVALID` | `NPS-CLIENT-BAD-PARAM` | Requested session validity is below 60 seconds or above the CA's configured maximum. |
| `NIP-CA-JWS-INVALID` | `NPS-AUTH-UNAUTHENTICATED` | Group-JWS authorization on a session-issue request fails signature, header, or shape validation. |
| `NIP-CA-JWS-EXPIRED` | `NPS-AUTH-UNAUTHENTICATED` | Group-JWS `iat` outside the CA's clock-skew window. |
| `NIP-CERT-PARENT-REVOKED` | `NPS-AUTH-UNAUTHENTICATED` | A session NID's parent / group NID is revoked or expired (chain check, §3.6). |

## 4. SDK changes

| SDK | Phase 1 (this CR) | Notes |
|---|---|---|
| **.NET** (`NPS.NIP`) | Required | Reference impl: extends `NipCertRecord` / `INipCaStore` / `NipCaService`, adds `NipCaService.RegisterGroupAsync` and `IssueSessionAsync`, adds JWS verification helper, extends HTTP router, adds DB migration `002_orchestrator_session.sql`. |
| **Python / TypeScript / Java / Rust / Go** | Deferred | Tracked as follow-up tickets per the same parity model used for NPS-RFC-0002 / RFC-0003 / RFC-0004. SDK clients can already _consume_ session IdentFrames via the existing IdentFrame DTO and the unknown-field pass-through behavior; what's deferred is the `IssueSessionAsync` client-side helper. |

## 5. Conformance changes

This CR does not add new conformance test items to `spec/services/conformance/NPS-Node-L1.md` — the L1 suite is Node-side and the group/session model is CA-server-side. Nodes verifying CR-0003 IdentFrames MUST do step 3a (chain check) per §7; this is added to the Node L1 suite as a follow-up under the next test-suite revision.

CA-side correctness is enforced by the unit test suite added to `impl/dotnet/tests/NPS.Tests/Nip/`:

- `OrchestratorGroupSessionTests.IssuingGroup_ReturnsLineageGroupFrame`
- `OrchestratorGroupSessionTests.IssuingSession_ChainsToGroup`
- `OrchestratorGroupSessionTests.IssuingSession_WithoutAuthority_IsRejected`
- `OrchestratorGroupSessionTests.LineageMetadata_IsSigned_AndVerifies`
- `OrchestratorGroupSessionTests.RevokingGroup_BlocksFutureSessions_AndCascades`

## 6. Migration impact

| Audience | Impact |
|---|---|
| Existing single-NID Operators | None. `register` / `renew` / `revoke` paths unchanged; lineage is opt-in. |
| Existing Verifiers (Nodes) | Must add chain-check (§3.6) to remain conformant when receiving session IdentFrames. Frames without `lineage` continue to verify unchanged. Until upgraded, a Node will *fail* to verify session IdentFrames (signature mismatch on unknown field) — this is a safe-fail (deny rather than admit). |
| CA Server operators | Must apply DB migration `002_orchestrator_session.sql` before upgrading the binary. |

## 7. Out of scope

- **Multi-level lineage chains** (group → sub-group → session). The `parent_nid` / `group_nid` split anticipates this but the CA enforces a 1-level chain in this CR; deeper chains are a follow-up.
- **Cross-CA delegation**. Sessions can only be issued by the same CA that issued the group; cross-CA session issuance lives under TrustFrame (NIP §5.2) and the commercial NPS Cloud surface.
- **Reputation log entries linked by group**. NIP §5.1.2 entries can already use `subject_nid = group_nid` to fan-in by orchestrator; no new entry shape is added here.
- **Per-session keypair attestation hardware**. Sessions get fresh Ed25519 keypairs but no requirement that they be HSM-resident; that's deployment policy.
- **ACME `agent-01`-style challenge for session issuance**. Sessions are CA-internal; ACME is for external-domain proof and is overkill here. The group-JWS path serves the same anti-replay role.

## 8. Acceptance criteria

- [ ] `spec/NPS-3-NIP.md` updated (§3, §5.1 lineage table, §5.3 reason enum, §7 step 3a, §8 endpoints, §9 errors, §11 changelog v0.7).
- [ ] `spec/error-codes.md` and `spec/NPS-3-NIP.cn.md` mirrors updated.
- [ ] `spec/cr/README.md` index updated.
- [ ] DB migration `tools/nip-ca-server/db/002_orchestrator_session.sql`.
- [ ] `NipCertRecord` carries `NidRole` / `ParentNid` / `LineageJson`.
- [ ] `INipCaStore.GetByParentNidAsync` implemented in InMemory / SQLite / PostgreSQL stores.
- [ ] `NipCaService.RegisterGroupAsync` / `IssueSessionAsync` / cascading `RevokeAsync` / chain-checking `VerifyAsync`.
- [ ] HTTP routes: `register-group`, `sessions/issue`, `groups/{nid}/revoke`, `groups/{nid}/sessions`.
- [ ] xUnit suite covers all five required scenarios (§5).
- [ ] CHANGELOG.md / CHANGELOG.cn.md entries.

## 9. CHANGELOG entry

```
- **NPS-CR-0003**: Orchestrator group NIDs and short-lived session NIDs.
  Adds `IdentFrame.lineage` (signed), reserves `group-` / `session-`
  identifier prefixes, adds the `parent_revoked` revocation reason and
  seven new error codes, and ships the CA endpoints
  `/v1/orchestrators/groups/{register, sessions/issue, revoke, list}`.
  Backward-compatible for ordinary single-NID registration; opt-in for
  orchestrators. Reference impl: .NET (`NPS.NIP`); other SDKs deferred.
```

## 10. Open questions

- **OQ-1** Should `lineage.session_id` duplicate the URN identifier, or be a freer field? Current draft requires it match the URN's `session-...` segment for indexability without URN parsing.
- **OQ-2** Should the cascade revocation also immediately push RevokeFrames into a (future) reputation log entry of `incident = "cert-revoked"`? Tentatively yes but deferred to RFC-0004 follow-up.
- **OQ-3** Do we need a per-group rate limit on `sessions/issue`? Not required by this CR; deployments handle it. RFC notes the threat (a stolen group key issuing thousands of sessions before detection).
