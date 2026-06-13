English | [中文版](./NPS-RFC-0005-reputation-policy-enforcement.cn.md)

---
**RFC Number**: NPS-RFC-0005
**Title**: Reputation Policy Enforcement
**Status**: Active
**Author(s)**: Ori Lynn <iamzerolin@gmail.com> (LabAcacia)
**Shepherd**: Ori Lynn (pre-1.0 fast-track per `spec/cr/README.md`)
**Created**: 2026-05-19
**Last-Updated**: 2026-05-28
**Accepted**: 2026-05-19 (pre-1.0 fast-track; see `spec/cr/README.md`)
**Activated**: 2026-05-28 (v1.0.0-alpha.9 — `DefaultReputationPolicyEvaluator`, `AnchorNodeMiddleware` enforcement, and `X-NWP-Ident` assurance extraction all shipped)
**Supersedes**: _none_
**Superseded-By**: _none_
**Affected Specs**: NPS-2 NWP, NPS-3 NIP, spec/services/NPS-AaaS-Profile.md, spec/error-codes.md
**Affected SDKs**: .NET, Python, TypeScript, Java, Rust, Go
---

# NPS-RFC-0005: Reputation Policy Enforcement

## 1. Summary

Define the wire format and enforcement lifecycle for `reputation_policy` — the
per-Node configuration that controls how an Anchor Node consults NPS-RFC-0004
reputation logs to make **accept / throttle / reject / ban** admission decisions
on incoming Agent requests.

This RFC converts the sketch in RFC-0004 §4.4 into a normative, implementable
specification, adds the corresponding `IdentFrame.metadata` field for Agent-side
log endorsements, and introduces the `ReputationPolicyEvaluator` interface that
SDK middleware must implement.

## 2. Motivation

RFC-0004 defines the log wire format but deliberately leaves enforcement
node-local. The AaaS Profile L2-09 requirement (SHOULD consult a log on
admission) lacks a standardized policy schema, making interoperability
between Node implementations and tooling (NPS Probe, NPS Studio) impossible
without a normative format.

Concrete gaps this RFC closes:

- **No shared policy schema**: two AaaS operators configuring "reject agents
  with major rate-limit violations" currently express this in incompatible
  ad-hoc formats, preventing log operators from publishing conformance claims.
- **No standard error codes for reputation decisions**: the existing
  `NWP-AUTH-REPUTATION-BLOCKED` from RFC-0004 is a single undifferentiated
  error; throttle vs. reject vs. ban require distinct responses so callers can
  implement appropriate retry logic.
- **No agent-side log endorsement**: Agents have no standard way to declare
  which log operators they accept queries from, making federated log discovery
  ambiguous.

## 3. Non-Goals

- Does NOT define what constitutes a reportable incident (that is RFC-0004 §4.2).
- Does NOT mandate a global canonical log or log federation scheme.
- Does NOT specify how Nodes cache log query results beyond normative minimums.
- Does NOT extend the `IdentFrame` signature to cover `metadata.reputation_policy`
  — metadata remains unsigned per NIP §5.1 trust boundary rules.

## 4. Detailed Design

### 4.1 Node-side policy: `reputation_policy` in NWM / `AnchorNodeOptions`

An Anchor Node MAY publish a `reputation_policy` block in its NWM manifest
(under the top-level `reputation_policy` key). The same structure drives the
`AnchorNodeOptions.ReputationPolicy` SDK configuration object.

#### 4.1.1 Wire format (JSON / YAML)

```json
{
  "reputation_policy": {
    "enabled": true,
    "log_sources": [
      "https://log.nps.example.com/v1/reputation",
      "https://consortium-log.nps.org/v1/reputation"
    ],
    "min_assurance_level": "attested",
    "cache_ttl_seconds": 300,
    "ban_ttl_seconds": 3600,
    "throttle_on": [
      { "incident": "rate-limit-violation", "severity": ">=minor", "within_days": 7 }
    ],
    "reject_on": [
      { "incident": "tos-violation",   "severity": ">=major",    "within_days": 30 },
      { "incident": "scraping-pattern","severity": ">=major",    "within_days": 30 }
    ],
    "ban_on": [
      { "incident": "cert-revoked",    "severity": ">=minor" },
      { "incident": "fraud",           "severity": ">=major" }
    ]
  }
}
```

#### 4.1.2 Field definitions

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `enabled` | bool | no | `true` | When `false`, the policy block is parsed but enforcement is skipped. Useful for dry-run auditing. |
| `log_sources` | array of URLs | yes (if enabled) | — | Ordered list of RFC-0004 compliant log operator base URLs. Queried in order; first successful response wins. |
| `min_assurance_level` | string | no | `"anonymous"` | Minimum RFC-0003 assurance level. One of `"anonymous"` / `"attested"` / `"verified"`. Requests from Agents below this level are rejected with `NWP-ASSURANCE-MISMATCH` before log queries are performed. |
| `cache_ttl_seconds` | uint32 | no | `300` | How long to cache a log query result per NID. 0 = no caching (query every request). |
| `ban_ttl_seconds` | uint32 | no | `3600` | How long a NID remains banned after a `ban_on` rule fires. Ban state is in-process; a Node restart clears bans. |
| `throttle_on` | array of rules | no | `[]` | Rules that trigger rate-limiting (HTTP 429). All rules are evaluated; any match triggers throttle. |
| `reject_on` | array of rules | no | `[]` | Rules that trigger a hard rejection (HTTP 403). Evaluated after `ban_on`; any match triggers rejection. |
| `ban_on` | array of rules | no | `[]` | Rules that trigger a time-limited ban (HTTP 403 + ban cache entry). Evaluated first. |

#### 4.1.3 Rule format

Each rule in `throttle_on`, `reject_on`, and `ban_on` is an object:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `incident` | string | yes | RFC-0004 §4.2 incident type, or `"*"` to match any incident. |
| `severity` | string | yes | Severity predicate. Format: `">=<level>"` or exact `"<level>"`. Severity scale (lowest to highest): `info` / `minor` / `moderate` / `major` / `critical`. |
| `within_days` | uint32 | no | If set, only considers log entries with `timestamp` within this many days of now. Absent = all time. |
| `count` | uint32 | no | Minimum number of matching entries required to fire the rule. Default 1. |

**Severity predicate**: `">=moderate"` matches `moderate`, `major`, and `critical`.
Exact form `"major"` matches only `major`.

#### 4.1.4 Enforcement order

The evaluator MUST apply rules in this order:

1. **`min_assurance_level` check** — if Agent's assurance level (from verified
   IdentFrame) is below the configured minimum, reject immediately with
   `NWP-ASSURANCE-MISMATCH`. Log queries are skipped.
2. **Ban cache check** — if the requester NID is in the local ban cache and
   the ban TTL has not expired, reject with `NWP-REPUTATION-BANNED` without
   querying the log.
3. **Log query** — query `log_sources` for the requester NID's entries. Use
   cached results if within `cache_ttl_seconds`. On log unavailability:
   implementations MAY fall back to the previous cached result or apply a
   configurable `on_log_unavailable` policy (`allow` | `deny`; default `allow`).
4. **`ban_on` evaluation** — if any rule fires, insert NID into ban cache for
   `ban_ttl_seconds`, then reject with `NWP-REPUTATION-BANNED`.
5. **`reject_on` evaluation** — if any rule fires, reject with
   `NWP-REPUTATION-REJECTED`.
6. **`throttle_on` evaluation** — if any rule fires, return
   `NWP-REPUTATION-THROTTLED` with `Retry-After: 60` header.
7. **Accept** — set `X-NWP-Reputation-Status: clean` response header and
   proceed to normal request handling.

The evaluator MUST NOT short-circuit between steps 4–6 on partial matches;
each rule set is evaluated independently in order. A NID that matches both
`ban_on` and `reject_on` receives the ban outcome (step 4 wins).

### 4.2 Agent-side declaration: `reputation_policy` in `IdentFrame.metadata`

An Agent MAY include a `reputation_policy` object in `IdentFrame.metadata`
to declare which log operators it endorses and its consent preference for
reputation queries. This field is **unsigned** (per NIP §5.1 trust boundary
rules) and is therefore advisory only.

```json
{
  "reputation_policy": {
    "log_sources": [
      "https://log.nps.example.com/v1/reputation"
    ],
    "consent": "public"
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `log_sources` | array of URLs | Log operators the Agent endorses. Nodes MUST NOT treat this as an authoritative allowlist for their own queries — it is a hint for log discovery only. |
| `consent` | string | One of `"public"` (Agent consents to reputation queries by any Node) or `"private"` (Agent prefers queries be limited to direct counterparties). Advisory only; Nodes MAY query any log regardless of this field. |

**Trust boundary**: Nodes MUST NOT use `IdentFrame.metadata.reputation_policy`
to elevate trust, bypass `min_assurance_level` checks, or substitute for
Node-operator-configured `log_sources`. An Agent claiming a clean log in its
own metadata is not a conformance signal.

### 4.3 SDK interface: `ReputationPolicyEvaluator`

Each SDK MUST provide an evaluator type implementing the following contract
(expressed in .NET; other languages use idiomatic equivalents):

```csharp
/// <summary>
/// Evaluates an incoming request NID against the configured reputation_policy.
/// </summary>
public interface IReputationPolicyEvaluator
{
    /// <param name="requesterNid">NID extracted from X-NWP-Agent.</param>
    /// <param name="assuranceLevel">Verified assurance level from IdentFrame.</param>
    /// <param name="policy">The node-configured ReputationPolicy.</param>
    /// <param name="ct">Cancellation token.</param>
    /// <returns>
    ///   ReputationDecision with Outcome (Accept/Throttle/Reject/Ban)
    ///   and MatchedRule (null on Accept).
    /// </returns>
    Task<ReputationDecision> EvaluateAsync(
        string requesterNid,
        AssuranceLevel assuranceLevel,
        ReputationPolicy policy,
        CancellationToken ct = default);
}

public enum ReputationOutcome { Accept, Throttle, Reject, Ban }

public sealed record ReputationDecision(
    ReputationOutcome Outcome,
    ReputationPolicyRule? MatchedRule,
    string? ErrorCode);        // NPS error code string for non-Accept outcomes
```

`AnchorNodeMiddleware` calls `IReputationPolicyEvaluator.EvaluateAsync` early
in the request pipeline, after identity verification and before action dispatch.
The evaluator is injectable (DI / constructor parameter) to allow custom
implementations (e.g. caching backends, test stubs).

### 4.4 Error codes

| Error Code | HTTP Status | NPS Status | Description |
|------------|-------------|------------|-------------|
| `NWP-REPUTATION-THROTTLED` | 429 | `NPS-CLIENT-RATE-LIMITED` | Request rate-limited due to reputation policy match. Response MUST include `Retry-After: 60` header. |
| `NWP-REPUTATION-REJECTED` | 403 | `NPS-AUTH-FORBIDDEN` | Request rejected due to reputation policy match (`reject_on` rule). |
| `NWP-REPUTATION-BANNED` | 403 | `NPS-AUTH-FORBIDDEN` | Request rejected and NID temporarily banned (`ban_on` rule or active ban cache entry). Response SHOULD include `X-NWP-Ban-Expires` (Unix timestamp). |

Error response body (JSON):

```json
{
  "status": "NWP-REPUTATION-REJECTED",
  "message": "Request rejected: tos-violation (major) within 30 days",
  "matched_incident": "tos-violation",
  "matched_severity": "major"
}
```

Existing RFC-0004 error code `NWP-AUTH-REPUTATION-BLOCKED` is superseded by the
three codes above. Implementations MUST emit the new codes; the old code MAY be
retained as a deprecated alias for one alpha cycle.

### 4.5 NWM publication

When `reputation_policy.enabled` is `true`, the Node MUST publish the full
policy object (including `log_sources`) under the `reputation_policy` key in
the NWM manifest served at `/.nwm`. This allows callers to discover the Node's
enforcement posture before sending requests.

Nodes MUST NOT publish the `ban_cache` state or per-NID query results in the
NWM.

### 4.6 AaaS-Profile alignment

AaaS-Profile L2-09 is updated:

> L2-09 (updated by RFC-0005): SHOULD configure a `reputation_policy` with
> `enabled: true` in the NWM and at least one `log_sources` entry. The
> **minimum recommended policy** for L2 is:
> ```json
> {
>   "ban_on":    [{ "incident": "cert-revoked", "severity": ">=minor" }],
>   "reject_on": [
>     { "incident": "tos-violation",    "severity": ">=major", "within_days": 30 },
>     { "incident": "fraud",            "severity": ">=major" }
>   ],
>   "throttle_on": [
>     { "incident": "rate-limit-violation", "severity": ">=minor", "within_days": 7 }
>   ]
> }
> ```

## 5. Security Considerations

### 5.1 Log unavailability

When all configured `log_sources` are unreachable, the evaluator applies the
`on_log_unavailable` policy. The default `allow` is chosen to avoid a DoS
vector where an attacker takes down log operators to block all requests. High-
security deployments SHOULD set `on_log_unavailable: deny` and accept the
availability trade-off.

### 5.2 Cache poisoning

Log query results are cached per-NID. An attacker who can poison DNS to redirect
`log_sources` URLs could serve false "clean" results. Implementations MUST
verify TLS certificate chains for `log_sources` URLs and SHOULD pin the log
operator's public key when the log URL is under operator control.

### 5.3 Reputation laundering

A banned NID could obtain a new NID (fresh key pair + fresh CA enrollment) to
evade reputation enforcement. This is the standard Sybil problem for identity
systems. Mitigations:
- `min_assurance_level: "verified"` raises the cost of fresh NID acquisition.
- CA operators can apply account-level enrollment throttle independent of NID.
- Consortium log operators can correlate behavioral patterns across NIDs.

### 5.4 Privacy

Log queries reveal the requester NID to the log operator. This is intentional
for public-federated deployments. Nodes in `org-private` NDP profile SHOULD
operate a local log mirror or aggregate queries to reduce per-request disclosure.

## 6. Implementation Phasing

| Alpha | Scope |
|-------|-------|
| **alpha.8** | RFC-0005 spec (this document); `ReputationPolicyEvaluator` + `AnchorNodeOptions.ReputationPolicy` in all six SDKs; NWM `reputation_policy` publication; three new error codes |
| **alpha.9** | `IdentFrame.metadata.reputation_policy` agent-side declaration; AaaS-Profile L2-09 updated text; NPS Probe conformance check for NWM policy presence |

## 7. Open Questions

1. **`on_log_unavailable` default**: this RFC defaults to `allow` for availability
   reasons. Should the AaaS Profile L2 SHOULD require `deny` for certain incident
   types (e.g. always deny cert-revoked regardless of log availability, using a
   locally-cached revocation list)? A local revocation cache solves this without
   a hard `deny` default but adds implementation complexity.

2. **Ban persistence**: this RFC specifies that ban state is in-process and
   cleared on restart. Should `org-private` and `public-federated` registry
   profiles (NDP §7.3) require ban state to persist across restarts? A SQLite-
   backed ban store would parallel the `graph_seq` persistence requirement.

3. **Multi-log consensus**: when `log_sources` has multiple entries and they
   disagree (log A says clean, log B says major violation), this RFC takes
   the most restrictive result. Should the spec allow a `quorum` mode where
   `ceil(N/2)` logs must agree before a rule fires?

## 8. Change Log

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-05-19 | Initial draft |

---

*Copyright: LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
