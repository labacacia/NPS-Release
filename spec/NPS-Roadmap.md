English | [中文版](./NPS-Roadmap.cn.md)

# NPS Roadmap

**Version**: 0.6  
**Date**: 2026-06-11  
**Owner**: LabAcacia / INNO LOTUS PTY LTD  

---

## Cadence

Each phase breaks into three segments:

```
① Spec Sprint   (2 weeks)   — freeze every spec for this phase
② Impl Sprint   (6–8 weeks) — implementation, testing, documentation
③ Review Gate   (1 week)    — community / internal review before advancing
```

## Version convention

| Version tag | Meaning |
|-------------|---------|
| `v0.x-draft` | internal draft; breaking changes allowed |
| `v0.x-alpha` | public preview; API unstable |
| `v0.x-beta`  | feature-complete; external testing welcome |
| `v1.0`       | spec frozen; production-ready |

---

## Phase 0 — Spec Unification (2026 Q2) — ✅ done

**Goal**: establish the full NPS spec skeleton, unify the frame namespace and naming, produce a v0.2-draft for community comment.

- [x] `NPS-0-Overview.md` v0.3
- [x] `NPS-1-NCP.md` v0.6 (dual transport, configurable frame size, ErrorFrame)
- [x] `NPS-2-NWP.md` v0.8 (Node-published AnchorFrame, CGN, topology queries)
- [x] `NPS-3-NIP.md` v0.5 (metadata field, NPS status codes, X.509 NID prototype)
- [x] `NPS-4-NDP.md` v0.5
- [x] `NPS-5-NOP.md` v0.4
- [x] `frame-registry.yaml` v0.9 (with ErrorFrame 0xFE)
- [x] `error-codes.md` v1.0 (with NPS status-code mapping, NIP cert error codes, NWP topology error codes)
- [x] `status-codes.md` v0.2 (NPS native status codes + HTTP mapping)
- [x] `token-budget.md` v0.2 (CGN metering + tokenizer resolution chain)
- [x] `services/NPS-AaaS-Profile.md` v0.4 (Anchor/Bridge Node, VPL, L1/L2/L3, NPS-CR-0002)
- [x] `services/NPS-Node-Profile.md` v0.1 (L1/L2/L3 + activation modes)
- [x] `services/conformance/NPS-Node-L1.md` v0.1 (21 TC-N1-* test cases)
- [x] `services/conformance/NPS-Node-L2.md` v0.1 (10 TC-N2-* test cases, topology queries)
- [x] LabAcacia repos public, Discussions enabled; `NPS-Dev` monorepo intentionally private
- [x] Published: alpha.1 (2026-04-10), alpha.2 (2026-04-19), alpha.3 (2026-04-26), alpha.4 (2026-04-30), alpha.5 (2026-05-01)

---

## Phase 1 — Core Implementation (2026 Q3) — ✅ shipped

**Goal**: NCP + NWP + NIP + NDP + NOP production-ready across six reference SDKs; NIP CA Server OSS in all six.

### SDKs

| Language   | Package                            | Status |
|------------|------------------------------------|--------|
| .NET       | `LabAcacia.NPS.Core` + `.NWP` + `.NWP.Anchor` + `.NWP.Bridge` + `.NIP` + `.NDP` + `.NOP` | ✅ v1.0.0-alpha.11 (655 tests) |
| Python     | `nps-lib` (PyPI)                   | ✅ v1.0.0-alpha.11 (211+ tests, ≥97% coverage) |
| TypeScript | `@labacacia/nps-sdk` (npm)         | ✅ v1.0.0-alpha.11 (284+ tests) |
| Java       | `com.labacacia.nps:nps-java` (Maven Central) | ✅ v1.0.0-alpha.11 (112+ tests) |
| Rust       | `nps-sdk` + 6 sibling crates (crates.io) | ✅ v1.0.0-alpha.11 (109 tests) |
| Go         | `github.com/labacacia/NPS-sdk-go`  | ✅ v1.0.0-alpha.11 (96 tests) |

> **Official complete-SDK set = these six languages** (.NET / Python / TypeScript / Java / Rust / Go).
> **C++ and PHP are de-scoped** as of alpha.12: tracked as *planned* placeholder stubs (`expected: stub` in `NPS-Release/version.yaml`), **not** part of the "all SDKs feature-aligned" claim and **not** a release gate. They re-enter scope only when source + tests exist. Public docs MUST NOT list C++/PHP among shipped SDKs.

### NIP CA Server (six-language reference deployment)

| Language   | Stack                           | Status |
|------------|---------------------------------|--------|
| C# / .NET  | ASP.NET Core + SQLite + Docker  | ✅ v0.1 |
| Python     | FastAPI + SQLite + Docker       | ✅ v0.1 |
| TypeScript | Fastify + SQLite + Docker       | ✅ v0.1 |
| Java       | Spring Boot 3.4 + SQLite        | ✅ v0.1 |
| Rust       | Axum + SQLite + Docker          | ✅ v0.1 |
| Go         | net/http stdlib + SQLite        | ✅ v0.1 |

### .NET server runtime (reference)

- [x] `NPS.Core` — frame codec, AnchorCache, EXT header
- [x] `NPS.NWP` — Memory Node middleware (SQL Server / PostgreSQL), 284 integration tests
- [x] `NPS.NWP.Anchor` — `IAnchorTopologyService` + `topology.snapshot` / `topology.stream` (NPS-CR-0002)
- [x] `NPS.NOP` — DAG validator + orchestration engine, delegation-chain depth limit, SSRF protection, exponential backoff retry, 429 tests
- [x] `NPS.NIP` — CA library: keygen, issuance / revocation, OCSP, CRL

### RFCs and CRs shipped in Phase 1

- [x] **NPS-RFC-0001** — NCP connection preamble `b"NPS/1.0\n"` (Accepted, all 6 SDKs)
- [x] **NPS-RFC-0002 Phase A/B** — X.509 NID + ACME `agent-01` prototype (Draft, all 6 SDKs; IANA PEN pending)
- [x] **NPS-RFC-0003** — Agent identity assurance levels `anonymous`/`attested`/`verified` (Accepted, all 6 SDKs)
- [x] **NPS-RFC-0004 Phase 1+2** — NID reputation log (CT-style); SQLite + RFC 9162 Merkle tree + operator-signed STH + inclusion proofs (`nps-ledger`)
- [x] **NPS-CR-0001** — Anchor/Bridge Node split; `NWP.Gateway` retired; `compat/*-ingress` renamed
- [x] **NPS-CR-0002** — Anchor Node topology queries; L2 conformance suite (10 tests)

### Daemons

| Daemon         | Status at alpha.5 |
|----------------|-------------------|
| `npsd`         | ✅ L1 + sub-NID issuance + per-NID inbox queue (17 integration tests) |
| `nps-registry` | ✅ SQLite-backed real registry (SqliteNdpRegistry, 10 tests) |
| `nps-ledger`   | ✅ Phase 3: SQLite + Merkle + STH + inclusion proofs + STH gossip (33 tests) |
| `nps-runner`   | Phase 1 skeleton (L3 runtime deferred) |
| `nps-ingress`  | Phase 1 skeleton (Internet ingress deferred) |
| `nps-cloud-ca` | Stubbed (2027 Q1+) |

### Completion bar

- [x] SDK unit coverage ≥ 90 % across all six languages
- [x] Memory Node `QueryFrame` round-trip integration tests passing
- [x] NIP CA Server one-command Docker Compose boot documented
- [x] Token-savings baseline ≥ 30 % vs REST (actual: 45.0 %)
- [x] Wire-size baseline vs JSON (actual: 63.6 % aggregate reduction)

---

## Phase 2 — Ecosystem Expansion (2026 Q4) — 🚧 in progress

**Goal**: adapters to existing ecosystems (MCP, A2A, gRPC), richer SDK examples, Tier-2 MsgPack production hardening.

- [x] `compat/mcp-ingress/` — NWP Memory/Action/Complex Node ↔ MCP 2024-11-05 adapter (`LabAcacia.McpIngress` v1.0.0-alpha.11)
- [x] `compat/a2a-ingress/` — NOP `TaskFrame` ↔ A2A Task adapter (`LabAcacia.A2aIngress` v1.0.0-alpha.11)
- [x] `compat/grpc-ingress/` — NWP Memory/Action/Complex Node ↔ gRPC adapter (`LabAcacia.GrpcIngress` v1.0.0-alpha.11)
- [x] Tier-2 MsgPack wire-size benchmark (aggregate 63.6 % reduction vs JSON)
- [x] Token-savings benchmark (aggregate 45.0 % CGN reduction vs REST)
- [x] NOP orchestrator executes a 3-node DAG end-to-end
- [x] Claude Desktop talks to an NWP Memory Node through `mcp-ingress`
- [x] `NDP.ResolveFrame` resolves `nwp://` to a physical endpoint via DNS TXT — `resolve_via_dns` / `resolveWithDns` / `ResolveViaDns` across all six SDKs; injectable `DnsTxtLookup` interface; system resolver per language
- [x] **NPS Probe** (Agent Coder conformance CLI) — shipped v0.1 (alpha.10), v0.2 (alpha.11); 5 checks
- [ ] **NPS Studio** (human visual debugger) — not started; deferred to a later cycle (not in alpha.12 scope)

---

## alpha.5 Release — 2026-05-01 ✅

### Completed in alpha.5

| Item | Notes |
|------|-------|
| **NPS-RFC-0004 Phase 3** — STH gossip for `nps-ledger` | `GossipState` + `GossipService` + `GET /v1/log/gossip/sth`; 13 new tests |
| **AaaS-Profile L2-09** — default `reputation_policy` | SHOULD requirement; minimum recommended policy defined |
| **`NWP-RESERVED-TYPE-UNSUPPORTED`** in AnchorNodeMiddleware | HTTP 501; `NPS-SERVER-UNSUPPORTED` status code added |
| **`topology:read` capability gate** on AnchorNodeMiddleware | `AnchorNodeOptions.RequireTopologyCapability`; `X-NWP-Capabilities` header |
| **`cgn_est` per-event** on `TopologyEventEnvelope` | UTF-8/4 estimate per §7.2 (SHOULD) |
| **AssuranceLevel `from_wire("")`** fix | Python, TS, Java SDKs; `""` → Anonymous |
| **Spec / doc CN sync** | `error-codes.cn.md`, `RFC-0004.cn.md`, `status-codes.cn.md` all up-to-date |

### Deferred to alpha.6

| Item | Notes |
|------|-------|
| **NPS-CR-0002 Phase 2** — server-side Anchor middleware push of topology updates | .NET reference push/notify now lands through `AnchorNodeMiddleware` + `IAnchorTopologyService`; non-.NET ports remain below |
| Non-.NET port of NPS-CR-0002 `AnchorNodeClient` topology client | .NET reference done; Python/TS/Go/Java/Rust need port |
| Non-.NET port of NPS-RFC-0004 reputation helpers (`ReputationLogClient`) | .NET reference done; port all six SDKs |
| Non-.NET port of NPS-RFC-0003 assurance-level enforcement helpers | Wired in .NET; other SDKs have enum only, no enforcement helpers |
| **NPS-RFC-0002** promotion Draft → Proposed/Accepted | Closed by NPS-CR-0004 (2026-05-08): IANA PEN **65715** assigned; OID arc `1.3.6.1.4.1.65715` replaces provisional `1.3.6.1.4.1.99999`; RFC-0002 promoted Draft → Proposed (wire-in lands in alpha.6) |

## alpha.6 Release — 2026-05-12 ✅

| Item | Notes |
|------|-------|
| **NPS-CR-0002 Phase 2** — server-side Anchor middleware push | .NET reference: topology push/notify through `AnchorNodeMiddleware` + `IAnchorTopologyService`; closes the `node_kind` compatibility window, requires `topology.filter.node_roles` |
| **NPS-RFC-0002** wire-in (IANA PEN 65715) | NPS-CR-0004 (2026-05-08) assigned IANA PEN **65715**; OID arc `1.3.6.1.4.1.65715` replaces provisional `1.3.6.1.4.1.99999`; RFC-0002 promoted Draft → Proposed |
| **`NDP.ResolveFrame` DNS TXT resolution** | `nwp://` → physical endpoint in all six SDKs — `resolve_via_dns` / `resolveWithDns` / `ResolveViaDns`; injectable `DnsTxtLookup` |
| **NPS-RFC-0003 assurance enforcement** | All six SDKs gain full `AssuranceLevel` enum + enforcement logic (previously enum-only outside .NET) |

---

## alpha.7 Release — 2026-05-17 ✅

| Item | Notes |
|------|-------|
| **NPS-CR-0002 `AnchorNodeClient`** (5 non-.NET SDKs) | `get_snapshot` + `subscribe` (stream / async-generator / channel per language) + topology data types (MemberInfo, TopologySnapshot, TopologyFilter, TopologyEvent) |
| **NPS-CR-0005** — NIP CA RA model (.NET reference) | `EnrollmentTier` enum, `Ca/Ra/` policy + store layer, 4 enrollment endpoints, 4 new error codes; `db/003_ra_model.sql` PostgreSQL migration |
| **CGN profile conversion spec (#51)** | `cgn-profiles.yaml` expanded with Google Gemini, Meta Llama, Mistral; `token-budget.md` §2.3 updated |
| **NWP + NOP OpenTelemetry instrumentation** | `ActivitySource` + `System.Diagnostics.Metrics` in NPS-sdk-dotnet; closes NPS-sdk-dotnet#5 |
| **NPS-RFC-0002** promotion Proposed → Accepted | OQ-3 resolved (deferred to follow-up RFC); no open questions remain |

> Carried into later alphas: NPS-RFC-0004 `ReputationLogClient` full client (Phase 2 Merkle / STH / inclusion proofs) across all six SDKs — .NET had Phase 1 data types only at alpha.7.

---

## alpha.8 Release — 2026-05-22 ✅

| Item | Notes |
|------|-------|
| **cgn_limit enforcement** (NWP AnchorNodeMiddleware) | Pre-execution check; `NWP-CGN-LIMIT-EXCEEDED` → 402; published in NWM `token_budget.cgn_limit`. All 6 SDKs (Python, TS, Go incomplete — carried to alpha.9 for non-.NET) |
| **RFC-0005 ReputationPolicyEvaluator** | `IReputationPolicyEvaluator`, `DefaultReputationPolicyEvaluator` (in-process ban cache + per-NID log-query cache); `AnchorNodeOptions.ReputationPolicy`; three new error codes + two response headers; NWM `reputation_policy` publication |
| **RFC-0005 Python + Go + TS ports** | cgn_limit + RFC-0005 reputation wired into Python, TypeScript, Go Anchor servers |
| **SubscribeFrame (0x12)** | Added to `NPS.NWP`; subscription lifecycle types |
| **NPS-CR-0005 RA model** (.NET) | Three-tier enrollment; 4 CA endpoints; `db/003_ra_model.sql` |
| **RFC-0002 → Accepted, RFC-0005 → Accepted** | Status promotions |

---

## alpha.9 Release — 2026-05-25 ✅

| Item | Notes |
|------|-------|
| **NOP Saga compensation** (NPS-5 v0.5) | `compensate_action` / `compensate_params_mapping` on `DagNode`; `compensation_policy` on `TaskFrame`; reverse-DAG rollback in `NopOrchestrator`; 2 new error codes |
| **NDP v0.7 AnnounceFrame fields** | 6 new fields: `activation_mode`, `node_roles`, `cluster_anchor`, `spawn_spec_ref`, `bridge_protocols`, `activation_endpoint` |
| **NDP security profiles** | `local-dev` / `org-private` / `public-federated`; IP-range enforcement in `InMemoryNdpRegistry`; ephemeral TTL cap (60 s) |
| **NPS-SDK-dotnet alpha.9** | All 10 packages on NuGet.org + Nexus |

---

## alpha.10 Release — 2026-05-28 ✅

| Item | Notes |
|------|-------|
| **IdentFrame assurance extraction** | `AnchorNodeMiddleware` parses `X-NWP-Ident` header → `AssuranceLevel` (RFC-0003 Phase 2); falls back to `Anonymous` on parse failure |
| **AssuranceHintUrl** | `AnchorNodeOptions.AssuranceHintUrl`; included in `NWP-AUTH-ASSURANCE-TOO-LOW` response |
| **IdentReputationPolicyHint** (RFC-0005 §4.2) | `IdentMetadata.reputation_policy` — unsigned advisory hint; `log_sources` + `consent` fields |
| **NPS Probe v0.1 CLI** | `tools/nps-probe/` — 4 checks (NWM reachable, `reputation_policy`, `token_budget`, log operator `/sth`); PASS/WARN/FAIL + `--json` output |
| **Spec status advances** | CR-0003 → Implemented; RFC-0004 → Active; RFC-0005 → Active |

---

## alpha.11 Release — 2026-05-28 ✅

> **Release rule (all future releases)**: every alpha MUST advance all five protocols (NCP / NWP / NIP / NDP / NOP) — both spec bump and SDK implementation. Releases that touch fewer than five protocols are held back until all five have substantive content.

| Item | Notes |
|------|-------|
| **NCP v0.7** | `max_concurrent_streams` negotiation (HelloFrame/CapsFrame, uint16, default 32, `NCP-STREAM-LIMIT-EXCEEDED`); QUIC stream mapping (one bidirectional stream per channel; HelloFrame on stream 0); rekeying at 2^32 frames or 24 h (`EXT rekey: true`, `NCP-REKEY-REQUIRED`); mid-stream ErrorFrame MAY→MUST |
| **NWP v0.13** | §13 SubscribeFrame formal spec (CR-0006 Accepted): `subscription_id`, filter, `heartbeat_interval_ms`, `max_events`, opaque `cursor`; `topology:subscribe` SHOULD→MUST in §12.4; NWM `trust_anchors` (CA NID URN array); `bridge_target` schema standardized |
| **NIP v0.9** | `IdentFrame.ocsp_staple` (base64url DER, `NIP-OCSP-STAPLE-EXPIRED`); OID table: `id-nps-node-roles` 65715.2.2 (ASN.1 SEQUENCE OF UTF8String) + `id-nps-capabilities` 65715.2.3; Phase 3 flag day at v1.0.0-beta.1 |
| **NDP v0.8** | GraphFrame §5 topology-snapshot format (graph_id / nodes / edges / ttl / metadata; max 256 nodes / 1024 edges; `NDP-GRAPH-INVALID`, `NDP-GRAPH-TOO-LARGE`); §9 federation forwarding (`ndp-forwarded-by`, max 3 hops, `NDP-FEDERATION-LOOP`); `spawn_spec_ref` schema (OCI image, command, resource_limits) |
| **NOP v0.6** | AlignStream ack/NAK protocol (window_size=16, `ack_seq`/`nak_seq`, `NOP-STREAM-NAK`); `weighted_first_k` + `merge_all` aggregate strategies; `DelegateFrame.target_cluster_anchor` cross-cluster routing; webhook HMAC (`callback_secret`, `X-NPS-Signature`, `NOP-CALLBACK-HMAC-MISSING`) |
| **CR-0006** (Accepted 2026-05-28) | SubscribeFrame §13 formal spec; frame-registry promotion `proposed → stable` |
| **RFC-0006** (Draft) | NCP native-mode transport binding: TCP length-prefix framing, QUIC stream mapping, rekeying, `max_concurrent_streams` conformance |
| **SDK parity — all 6 SDKs** | Python/TS/Go/Java/Rust/.NET at alpha.11: NOP saga + AlignStream ack + cross-cluster; NDP security profiles + GraphFrame §5 + AnnounceFrame fields; NIP `ocsp_staple` + OID constants; NWP `SubscribeFrame` CR-0006 + `trust_anchors`; .NET: `IdNpsCapabilities` (65715.2.3), `TrustAnchors` in AnchorNodeOptions, `GraphFrame` §5 rewrite, `SubscribeFrame` CR-0006, `AggregateStrategy.WeightedFirstK/MergeAll`; 10 NuGet packages at 1.0.0-alpha.11 |
| **nps-ledger v1.0.0-alpha.11** | `POST /v1/log/federation/push` (NDP §9 loop detection, `X-NPS-Forwarded-By`, max 3 hops) |
| **nps-probe v0.2** | Check 5: NWM `trust_anchors` format validation (NWP v0.13 §4.1) |
| **nps-orchestrator v1.0.0-alpha.11** | Version bump + CHANGELOG backfill alpha.9/10/11 |
| **NPS-NWP-Manager v0.1** | Initial stub: `GET /health`, `GET /v1/nodes` (NWM fetch/cache), `GET /v1/nodes/list` |

---

## alpha.12 — 🚧 next (target 2026-06)

> **Theme**: *Parity & Edge* — bring the six reference SDKs to true **functional** parity (not just source presence), advance all five protocol specs, and stand up the L2/L3 daemon edge.
>
> Detailed implementation plan: [`docs/roadmap.md`](../docs/roadmap.md).

**Release gates** (all must ship before tagging):

1. **SDK functional parity** (hard gate) — close the gaps in `SDK_ALIGNMENT_ALPHA11`: port the full **Anchor/Bridge Node**, **CGN / token-budget**, and **reputation-policy** implementations from the .NET reference to Python / TypeScript / Java / Rust / Go. Source-presence is no longer sufficient; each language must pass an equivalent of the .NET Anchor/Bridge + CGN + reputation test suites.
2. **Five-protocol advancement** (release rule) — substantive spec + SDK content for every protocol:
   - **NCP v0.8** — promote **RFC-0006** (native-mode transport binding) Draft → Proposed; TLS binding for native mode (ALPN `nps/1.0`, mutual TLS), session resumption ticket. Couples with `nps-ingress` L2.
   - **NWP v0.14** — Bridge Node formal conformance section + `bridge_target` round-trip test vectors (parity-driven).
   - **NIP v0.10** — short-lived cert / renewal profile for edge mTLS; ties to `nps-ingress` certificate handling.
   - **NDP v0.9** — AnnounceFrame liveness/health field + resolve-time staleness check.
   - **NOP v0.7** — **NPS-CR-0007** (NOP ↔ L3 runtime integration): task-claim protocol, `spawn_spec_ref` semantics, idle/max-runtime reporting. Couples with `nps-runner` L3.
3. **Daemon L2/L3** — `nps-ingress` L2 (NCP over TLS, ALPN `nps/1.0`, mutual TLS, `:8080`→`:443` termination, L2 conformance TC-N2-*) + `nps-runner` L3 FaaS runtime (NOP task scheduler, worker lifecycle via `spawn_spec_ref`, sync-barrier coordination).
4. **C++/PHP de-scoped** — explicitly removed from the "official complete-SDK set"; tracked as *planned*, not blocking (see note under Phase 1 SDK table).

**Per-protocol deliverables** (concrete frames / fields / error codes):

| Item | Notes |
|------|-------|
| **NCP v0.8** | RFC-0006 native-mode TLS binding (ALPN `nps/1.0`, mTLS, session-NID binding `NCP-NID-MISMATCH`, resumption tickets; §7.5); NopFrame (0x07) keepalive/heartbeat (null payload, bidirectional); `HelloFrame.ping_interval_ms` (uint32, 0=disabled); `NCP-KEEPALIVE-TIMEOUT` error code (`NPS-SERVER-TIMEOUT`); §7.6 dead-peer detection at `3 × ping_interval_ms`; rekeying rule before 2^32 frames or 24 h; `NCP-REKEY-REQUIRED` |
| **NWP v0.14** | Bridge Node formal conformance (§16) + `bridge_target` round-trip vectors; NWM `manifest_version` type changed to uint32 monotonic counter; new NWM field `manifest_updated_at` (ISO 8601); `X-NWM-Version` response header MUST on all `GET /.nwm` responses; conditional-request via `If-None-Match: <uint32>` |
| **NIP v0.10** | §6.1 short-lived / renewable edge-mTLS cert profile; `IdentFrame.node_roles` (array[string]) self-declared Phase 1–2; Phase 3 CA-attested via `id-nps-node-roles` extension (65715.2.2); `NIP-CERT-NODE-ROLES-MISMATCH` error code |
| **NDP v0.9** | `AnnounceFrame` liveness fields `health` / `last_seen` + §3.2.1 resolve-time staleness `NDP-RESOLVE-STALE`; `heartbeat_interval_ms` (uint32, default 60000) + announce-time `NDP-ANNOUNCE-STALE`; `spawn_spec_ref` (string ref) resolving to a SpawnSpec, formal schema §3.1.2 (oci_image, command, resource_limits: cpu_millicores/memory_mb, Profile L3); §9 federation forwarding loop detection |
| **NOP v0.7** | **NPS-CR-0007** NOP ↔ L3 runtime (§8: task-claim lease, `NOP-CLAIM-CONFLICT`, `NOP-SPAWN-SPEC-INVALID`, `NOP-RUNTIME-IDLE-TIMEOUT`, `NOP-RUNTIME-MAX-RUNTIME`; conformance `TC-N3-*`); `TaskFrame.result_ttl_seconds` (uint32, default 3600), `NOP-TASK-RESULT-EXPIRED`; `NOP-STREAM-NAK-UNRESOLVABLE` for evicted-frame NAK; frame-registry: NopFrame 0x07 registered as stable |

---

## Phase 3 — Ecosystem Validation (2027 Q1–Q2)

**Goal**: real-world PoCs, NPS Cloud CA v1.0 live, lay the groundwork for de-facto standard status.

- [ ] NPS Cloud CA v1.0 (multi-region HA, real-time OCSP, Professional Plan)
- [ ] LangChain / AutoGen / CrewAI integration adapter packages
- [ ] FinTech PoC (Open Banking scenario, cross-org `TrustFrame`)
- [ ] Connected-vehicle PoC (device NIDs, `StreamFrame` real-time telemetry)
- [ ] Token-savings benchmark report (publicly released)
- [ ] NIP CA Server OSS v1.0 (PostgreSQL + Web Admin UI)
- [ ] NIP CA Server self-hosted on an NWP Memory Node backend (dogfooding)
- [ ] GitHub stars ≥ 500

---

## Phase 4 — Standardization (2027 Q3 onward)

**Goal**: push NPS toward formal W3C / IETF standardization; freeze NPS 1.0.

- [ ] Joint vendor support statement (≥ 3 vendors)
- [ ] W3C WebAI Community Group proposal
- [ ] IETF Internet-Draft (NCP + NWP core specs)
- [ ] NPS 1.0 spec freeze
- [ ] ISO/IEC JTC 1 evaluation
- [ ] Tier-3 MatrixTensor specification

---

## Milestone Dependency Graph

```
Phase 0                Phase 1                  Phase 2             Phase 3
──────                 ────────                 ────────            ────────
[spec skeleton]
    │
    ├──→ [NPS.Core] ──→ [NWP Memory/Action] ──→ [Complex Node]
    │         │                  │              [mcp-ingress] ──→ [framework integrations]
    │    [NIP CA OSS] ──────────────────────→  [a2a-ingress]
    │         │                               [grpc-ingress]
    └──→ [six SDKs] ─────────────────────────→ [SDK parity]
                                               [NDP DNS TXT]
                                               [NOP orchestr] [Cloud CA]──→ [PoC]
```

---

## Risk Register

| ID  | Risk | Probability | Impact | Mitigation |
|-----|------|-------------|--------|-----------|
| R01 | Spec changes force implementation rework | High | High | Phase 0 freezes spec before implementation; changes go through RFC |
| R02 | MCP ecosystem evolves quickly, breaks the ingress adapter | Medium | Medium | Version `mcp-ingress` independently |
| R03 | Token savings fall short (<30%) | Medium | High | Benchmark from Phase 1 (actual: 45 %); AnchorFrame hit rate is the lever |
| R04 | NIP CA private-key incident | Low | Critical | Reserve HSM interface; enforce annual key rotation |
| R05 | Competitor reaches similar positioning first | Medium | Medium | NPS differentiates on Token Economy; accelerate OSS release |
| R06 | Phase 3 PoC partner resources fall through | Medium | Medium | Backup: internal demo datasets in lieu of real partners |
| R07 | W3C/IETF cycle too long | High | Low | Pursue de-facto-standard path (GitHub adoption) before formal RFC |
| R08 | IANA PEN assignment delayed | Medium | Low | RFC-0002 ships with provisional OID; IANA PEN is non-blocking for alpha releases |

---

*Copyright: LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
