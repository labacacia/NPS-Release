English | [中文版](./NPS-Roadmap.cn.md)

# NPS Roadmap

**Version**: 0.9
**Date**: 2026-08-16
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

- [x] `NPS-0-Overview.md` v0.4
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
> **C++ and PHP are de-scoped** as of alpha.13: tracked as *planned* placeholder stubs (`expected: stub` in `NPS-Release/version.yaml`), **not** part of the "all SDKs feature-aligned" claim and **not** a release gate. They re-enter scope only when source + tests exist. Public docs MUST NOT list C++/PHP among shipped SDKs.

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
- [ ] **NPS Studio** (human visual debugger) — not started; deferred to a later cycle (not in alpha.13 scope)

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
| **NDP v0.8** | GraphFrame §3.3 topology-snapshot format (graph_id / nodes / edges / ttl / metadata; max 256 nodes / 1024 edges; `NDP-GRAPH-INVALID`, `NDP-GRAPH-TOO-LARGE`); §9 federation forwarding (`ndp-forwarded-by`, max 3 hops, `NDP-FEDERATION-LOOP`); `spawn_spec_ref` schema (OCI image, command, resource_limits) |
| **NOP v0.6** | AlignStream ack/NAK protocol (window_size=16, `ack_seq`/`nak_seq`, `NOP-STREAM-NAK`); `weighted_first_k` + `merge_all` aggregate strategies; `DelegateFrame.target_cluster_anchor` cross-cluster routing; webhook HMAC (`callback_secret`, `X-NPS-Signature`, `NOP-CALLBACK-HMAC-MISSING`) |
| **CR-0006** (Accepted 2026-05-28) | SubscribeFrame §13 formal spec; frame-registry promotion `proposed → stable` |
| **RFC-0006** (Draft) | NCP native-mode transport binding: TCP length-prefix framing, QUIC stream mapping, rekeying, `max_concurrent_streams` conformance |
| **SDK parity — all 6 SDKs** | Python/TS/Go/Java/Rust/.NET at alpha.11: NOP saga + AlignStream ack + cross-cluster; NDP security profiles + GraphFrame topology snapshots + AnnounceFrame fields; NIP `ocsp_staple` + OID constants; NWP `SubscribeFrame` CR-0006 + `trust_anchors`; .NET: `IdNpsCapabilities` (65715.2.3), `TrustAnchors` in AnchorNodeOptions, `GraphFrame` topology-snapshot rewrite, `SubscribeFrame` CR-0006, `AggregateStrategy.WeightedFirstK/MergeAll`; 10 NuGet packages at 1.0.0-alpha.11 |
| **nps-ledger v1.0.0-alpha.11** | `POST /v1/log/federation/push` (NDP §9 loop detection, `X-NPS-Forwarded-By`, max 3 hops) |
| **nps-probe v0.2** | Check 5: NWM `trust_anchors` format validation (NWP v0.13 §4.1) |
| **nps-orchestrator v1.0.0-alpha.11** | Version bump + CHANGELOG backfill alpha.9/10/11 |
| **NPS-NWP-Manager v0.1** | Initial stub: `GET /health`, `GET /v1/nodes` (NWM fetch/cache), `GET /v1/nodes/list` |

---

## alpha.13 — 2026-06-13 ✅ (Parity & Edge; delivered across alpha.13–15)

> **Theme**: *Parity & Edge* — bring the six reference SDKs to true **functional** parity (not just source presence), advance all five protocol specs, and stand up the L2/L3 daemon edge.
>
> **Outcome (shipped in full across alpha.13–15)**: six-SDK functional parity; all five protocol specs advanced (NCP v0.9 / NWP v0.14 / NIP v0.10 / NDP v0.9 / NOP v0.7); and the L2/L3 daemon edge — `nps-ingress` NCP-over-TLS terminator (`NcpTlsListener`, ALPN `nps/1.0`, mTLS, `NCP-NID-MISMATCH` session-NID binding) and `nps-runner` L3 (CR-0007 lease + renewal, `SpawnSpec`, worker lifecycle). Remaining finishing touch — full `TC-N2-*` conformance vectors — carries into alpha.16.
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
| **NCP v0.9** | Tier-3 BinaryVector v1 (`binary_vector.v1`, `NPBV` payload, MessagePack metadata + little-endian float32 segments) for NWP vector search; RFC-0006 native-mode TLS binding (ALPN `nps/1.0`, mTLS, session-NID binding `NCP-NID-MISMATCH`, resumption tickets; §7.5); NopFrame (0x07) keepalive/heartbeat (null payload, bidirectional); `HelloFrame.ping_interval_ms` (uint32, 0=disabled); `NCP-KEEPALIVE-TIMEOUT` error code (`NPS-SERVER-TIMEOUT`); §7.6 dead-peer detection at `3 × ping_interval_ms`; rekeying rule before 2^32 frames or 24 h; `NCP-REKEY-REQUIRED` |
| **NWP v0.14** | Bridge Node formal conformance (§16) + `bridge_target` round-trip vectors; NWM `manifest_version` type changed to uint32 monotonic counter; new NWM field `manifest_updated_at` (ISO 8601); `X-NWM-Version` response header MUST on all `GET /.nwm` responses; conditional-request via `If-None-Match: <uint32>` |
| **NIP v0.10** | §6.1 short-lived / renewable edge-mTLS cert profile; `IdentFrame.node_roles` (array[string]) self-declared Phase 1–2; Phase 3 CA-attested via `id-nps-node-roles` extension (65715.2.2); `NIP-CERT-NODE-ROLES-MISMATCH` error code |
| **NDP v0.9** | `AnnounceFrame` liveness fields `health` / `last_seen` + §3.2.1 resolve-time staleness `NDP-RESOLVE-STALE`; `heartbeat_interval_ms` (uint32, default 60000) + announce-time `NDP-ANNOUNCE-STALE`; `spawn_spec_ref` (string ref) resolving to a SpawnSpec, formal schema §3.1.2 (oci_image, command, resource_limits: cpu_millicores/memory_mb, Profile L3); §9 federation forwarding loop detection |
| **NOP v0.7** | **NPS-CR-0007** NOP ↔ L3 runtime (§8: task-claim lease, `NOP-CLAIM-CONFLICT`, `NOP-SPAWN-SPEC-INVALID`, `NOP-RUNTIME-IDLE-TIMEOUT`, `NOP-RUNTIME-MAX-RUNTIME`; conformance `TC-N3-*`); `TaskFrame.result_ttl_seconds` (uint32, default 3600), `NOP-TASK-RESULT-EXPIRED`; `NOP-STREAM-NAK-UNRESOLVABLE` for evicted-frame NAK; frame-registry: NopFrame 0x07 registered as stable |

---

## alpha.14 — 2026-06-26 ✅

| Item | Notes |
|------|-------|
| **NCP Tier-3 BinaryVector — SDK impl** | `binary_vector.v1` codec across the six SDK source trees + malformed-frame/client-error conformance coverage (spec landed alpha.13) |
| **Inbound NWP Bridge server adapters** | `McpInboundServer` / `A2aInboundServer` / `GrpcInboundService` + host `BridgeServerHandler` / `BridgeServerApp` — external MCP/A2A/gRPC clients invoke fronted NPS nodes; secure-by-default (NID + verifier, bounded body, dispatch timeout, sanitized errors) |
| **Native-mode NWP serving** | `NwpNativeNodeServer` serves `QueryFrame`/`ActionFrame` over an `NcpSession` |
| **Typed remote NIP CA client** | `NipCaClient` (discovery, CRL, register/renew/revoke/verify, RFC-0002 X.509); `/v1/crl` gains `issued_at` + detached CA signature |
| **Daemon observability + conformance harness** | transport-neutral `HealthProbeRenderer`; `LabAcacia.NPS.Conformance` (Node L1/L2 catalogs) |

## alpha.15 — 2026-06-28 ✅

> **Theme**: *Consistency* — cross-SDK wire correctness.
>
> **Numbering note**: the `1.0.0-alpha.15` CHANGELOG heading kept accumulating after the tag — the
> LLM/Thinking Profile series filed under it (NWP v0.15–v0.17, NIP v0.11) is dated 2026-07-04/05 and
> did **not** reach any registry as alpha.15. It shipped as **alpha.16** (see the next section).
> The three rows below are what alpha.15 actually published.

| Item | Notes |
|------|-------|
| **NIP TrustFrame/RevokeFrame signed-payload realignment** (breaking) | Signed payload aligned to the current NPS-3 fields (`issued_at`, `serial`, `signer_nid`, `target_nid`); `NIP-CERT-REVOKED` naming. Old alpha.14-era signed frames no longer verify |
| **NDP AnnounceFrame signed canonical form — normative & cross-SDK consistent** (breaking) | Signed body excludes `signature`/`health`/`last_seen`/`frame`; null optionals omitted; `heartbeat_interval_ms` default `60000` only when absent, explicit `0` signed literally. Aligned across all six SDKs |
| **NDP graph + NIP revoke guard enforcement** | GraphFrame 256-node/1024-edge bounds (`NDP-GRAPH-TOO-LARGE`), 3-hop federation loop (`NDP-FEDERATION-LOOP`), RevokeFrame `parent_nid`↔`parent_revoked` rule |

---

## alpha.16 — 2026-07-23 ✅

> **Theme**: *LLM / Thinking Profile* — make model-serving Nodes first-class in the NWM, and close the
> HTTP-overlay error registry.
>
> **This is what alpha.16 actually published** — not the *HA & Hardening* content the earlier revision of
> this roadmap pencilled in under the alpha.16 heading. That content moved to the **alpha.17** section below.
>
> **Why the number**: the alpha.15 package numbers were already taken on the public registries, so this
> content — filed under the `1.0.0-alpha.15` CHANGELOG heading — was re-issued as alpha.16.
>
> **Published protocol set**: NCP v0.9 / NWP v0.17 / NIP v0.11 / NDP v0.9 / NOP v0.7; `error-codes.md` v1.6.

| Item | Notes |
|------|-------|
| **NWP v0.15 — `llm.complete` ActionFrame contract** | New §7.5: typed request/response DTO shape, `stop_reason` enum, tool-call field names, sync / async / streaming response semantics, ErrorFrame-vs-payload-error rule, snake_case JSON/MessagePack key policy. No new frame type or error code |
| **NWP v0.16 — NWM `profiles` + LLM/Thinking Profile** | New §4.2a `profiles.llm` for model-serving Action/Complex Nodes; "Thinking Node" is a product-facing alias, **not** a new `node_type`; coarse discovery via NIP/NDP `llm:*` capabilities, detailed model / streaming / tool / privacy / reasoning-disclosure metadata lives in the NWM. .NET DTOs + helpers shipped |
| **NIP v0.11 — `llm:*` capability registry** | `llm:complete`, `llm:stream`, `llm:tool_call`, `llm:embed`, `llm:rerank`; TrustFrame `trust_scope` may cover them. No new frame fields or error codes |
| **NWP v0.17 — HTTP binding rejection codes** | New §9.5 + `error-codes.md` v1.6: `NWP-HTTP-ORIGIN-FORBIDDEN`, `NWP-HTTP-CONTENT-TYPE-UNSUPPORTED`, `NWP-HTTP-ACCEPT-UNSATISFIABLE`, `NWP-HTTP-REQUEST-ID-MISMATCH`, `NWP-HTTP-FRAME-BODY-MALFORMED`, `NWP-CAPABILITY-ADVERTISED-UNIMPLEMENTED` (advertised-but-unimplemented rollout window). Closes NPS-Dev#84 |
| **NIP CA — RA store persistence** | Three-tier RA enrollment stores (NPS-CR-0005) persisted in the CA storage backends instead of memory-only |
| **Bridge schema fixes + daemon test isolation** | `bridge_target` payload-contract corrections; `nps-ingress` / `nps-runner` distribution builds no longer compile test sources into the application projects (internal coverage preserved via explicit test-assembly visibility) |

**Known defect shipped in alpha.16** (fixed on `main`, ships in alpha.17): the published **Go / Rust / TypeScript / Java** SDKs emit NOP `DelegateFrame` wire keys `task_id` / `target_nid`, where NPS-5 requires `parent_task_id` / `target_agent_nid` — verified against each SDK's `v1.0.0-alpha.16` tag. Rust additionally diverges on `SyncFrame.subtask_ids` (spec: `wait_for`) and `AlignStream.sync_id` / `source_nid` (spec: `stream_id` / `sender_nid`). .NET and Python are conformant, so **cross-language NOP delegation between a .NET/Python peer and a Go/Rust/TS/Java peer does not interoperate at alpha.16**.

---

## alpha.17 — 2026-08-03 ✅ — **released to all registries**

> **Theme**: *HA & Hardening + Bidirectional Bridge + Portable Server/Conformance Profiles* — a cluster survives Anchor loss ([NPS-CR-0009](cr/NPS-CR-0009-multi-anchor-ha.md)), the Bridge Node becomes a two-way protocol boundary ([NPS-CR-0010](cr/NPS-CR-0010-bridge-bidirectional.md)), NIP gains its Phase-3 enforcement switch ahead of the `v1.0.0-beta.1` flag day, the cross-SDK NOP wire-key defect shipped in alpha.16 is fixed, and every protocol gains a **portable server / conformance profile** so the six SDKs implement one another's server surface identically.
>
> **How this release was built — a merge of two parallel lines.** alpha.17 was developed concurrently in
> two workspaces and reconciled into one candidate. One line delivered CR-0009 multi-Anchor HA, CR-0010's
> bidirectional Bridge (the consolidated `bridge_inbound` server that **replaces** the earlier duplicate
> `McpServerBridge` / `A2aServerBridge`), NIP Phase-3, the `TC-N2-HA-*` vectors, the `nps-registry` cluster
> resolution, and the CR-0009 CN bodies. The other delivered the **portable server / conformance-profile
> wave** across all six SDKs (NWP §16.5, NCP §2.6.2, NDP registry conformance + `graph_seq`, NOP
> orchestrator profile, NIP §7.6). The union is green across all six languages.
>
> **Renumbering**: the CR-0009/0010 edge line first rebased onto the released alpha.16 numbers as NWP
> 0.18/0.19, NIP 0.12, NDP 0.10/0.11, NCP 0.10, NOP 0.8. The portable-profile wave then added **one further
> version to each** protocol: NCP **0.11**, NWP **0.20**, NIP **0.13**, NDP **0.12**, NOP **0.9**;
> `error-codes.md` **1.8**, `frame-registry.yaml` 0.14. Any reference to the smaller
> "0.18-0.19 / 0.12 / 0.10-0.11 / 0.10 / 0.8 / 1.7" set is the pre-profile-wave numbering and is stale.

**Release gates**:

1. **Spec content** — ✅ **done**. Five-protocol advancement: NCP v0.11 / NWP v0.20 / NIP v0.13 / NDP v0.12 / NOP v0.9; `error-codes.md` v1.8; `frame-registry.yaml` v0.14.
2. **Cross-SDK implementation parity** (hard gate — the standing release rule is *spec **and** SDK*) — ✅ **done**. CR-0009 `cluster_epoch` / failover, NIP `phase3_enforcement`, CR-0010's inbound servers (`McpInboundServer` / `A2aInboundServer` / `GrpcInboundService`), and the portable server / conformance profiles are all implemented across the six SDKs (Go / Java / Python / Rust / TypeScript / .NET), each with passing suites.
3. **NOP wire-key fix reaches the *published* packages** (hard gate) — ✅ **done**. `DelegateFrame` (`parent_task_id` / `target_agent_nid`), `SyncFrame` (`wait_for`), and `AlignStream` (`stream_id` / `sender_nid`) wire keys aligned to NPS-5 in all SDKs; TS/Java keep a legacy-key decode fallback.
4. **Conformance** — ✅ **done**. `TC-N2-BridgeIn-01..06` and `TC-N2-HA-01..09` (both EN and CN); `nps-registry` implements highest-epoch resolution + `NDP-CLUSTER-SPLIT`, so the HA family has a reference to run against.
5. **Distribution** — ✅ **done**. Dev was synchronized to `NPS-Release/spec`, all six standalone SDK repositories, daemon/tool distributions, documentation surfaces, and mirrors; `NPS-Release/version.yaml` was bumped last and alpha.17 was tagged and published.
6. **CN translation** — ✅ CR-0009 and CR-0010 bodies plus NIP §7.5 are translated; residual portable-profile bodies are tracked in `version-matrix.yaml` `translation_lag`.
7. **Status hygiene** — ✅ **RFC-0006** Accepted (native mode normative as of NCP v0.10); **CR-0008 / CR-0009 / CR-0010** → Implemented; `spec/rfcs/README.md` + `spec/cr/README.md` tables refreshed; `CLAUDE.md` current.

**Per-protocol deliverables** (all on the release candidate):

| Protocol | Version | Notes |
|---|---|---|
| **NCP** | v0.10 + v0.11 | **0.10**: RFC-0006 Accepted (native-mode transport normative); **session continuity across Anchor failover** (CR-0009) — on connection loss / `NCP-NID-MISMATCH` after an ownership transfer, the client re-resolves via NDP §9 (highest `cluster_epoch`) or the NWP `anchor_failover` `successor_nid`. **0.11**: §2.6.2 **Native Server Interoperability Profile** — authentication-before-preamble ordering, bounded preamble/Hello reads, allocation-safe Hello limits, silent pre-admission failure |
| **NWP** | v0.18 + v0.19 + v0.20 | **0.18 (CR-0009)**: `anchor_failover` (`successor_nid` / `cluster_epoch` / `reason`) and `anchor_quorum_lost` (`quorum_size` / `available`) finalised; `cluster_epoch` (uint64) ownership fence; `NWP-ANCHOR-NOT-LEADER`, `NWP-ANCHOR-EPOCH-FENCED`. Single-Anchor clusters stay at epoch 1. **0.19 (CR-0010)**: Bridge Node bidirectional — Outbound + Inbound MUST lists, MCP inbound MUST serve `resources/*` as well as `tools/*`; §16 split into outbound (§16.1.1) / inbound (§16.1.2) profiles + direction declaration (§16.2) + error-mapping tables (§16.3); `compat/*-ingress` absorbed into `NPS.NWP.Bridge`; `NWP-BRIDGE-DIRECTION-UNSUPPORTED`. **0.20**: §16.5 **portable Node/Bridge server profile** + shared cross-language vectors — standardized HTTP/native admission, role dispatch, canonical/legacy MIME handling, finite body limits |
| **NIP** | v0.12 + v0.13 | **0.12**: §7.5 **Phase-3 enforcement mode** — receiver-side `phase3_enforcement` flag turning the Phase-1–2 opt-in CA-attestation checks (assurance / `node_roles` / capabilities / OCSP staple) into hard MUSTs; subset checks, each active only when the matching cert extension is present; `NIP-CERT-CAPABILITIES-EXCEEDED`. **0.13**: §7.6 **Portable CA and Verification Profile** — deterministic verification/source order, `if_configured` and fail-closed `required` revocation modes, signed deterministic CRL semantics, full CA-store enumeration, authenticated `GET /v1/certificates` |
| **NDP** | v0.10 + v0.11 + v0.12 | **0.10 (CR-0009)**: `cluster_epoch` (uint64, default 1) on AnnounceFrame; §9 highest-epoch resolution, equal-epoch split → `NDP-CLUSTER-SPLIT`; federated propagation of `(cluster_anchor, cluster_epoch, active_nid)`. **0.11 (CR-0010)**: `bridge_inbound_protocols`; a `"bridge"` node MUST have at least one of the two arrays non-empty. **0.12**: **Registry Conformance profile** — additive `graph_seq` wire field + compatibility behavior; deterministic signed-body canonicalization; ordered signature/profile/replay/conflict/staleness checks |
| **NOP** | v0.8 + v0.9 | **0.8 (CR-0009)**: `DelegateFrame.target_cluster_anchor` MUST resolve to the cluster's current active Anchor (highest `cluster_epoch`); in-flight delegations MUST re-resolve to `successor_nid` on `anchor_failover` before retry; §8 lease-renewal semantics formalised (`NOP-CLAIM-CONFLICT`). **0.9**: **portable orchestrator profile** — deterministic DAG preflight and conformance scheduling; single-evaluation condition/input mapping; retry / timeout / cancellation / K-of-N / aggregate rules |
| **Implementation fix** | — | **NOP frame wire keys** aligned to NPS-5 in Go / Rust / TS / Java: `DelegateFrame` `task_id`→`parent_task_id`, `target_nid`\|`agent_nid`→`target_agent_nid`; `SyncFrame` `subtask_ids`→`wait_for`; `AlignStream` `sync_id`→`stream_id`, `source_nid`→`sender_nid`. Pure conformance fix; TS/Java keep a legacy-key decode fallback |
| **Shared** | error-codes v1.8 · frame-registry v0.14 | CR-0009's five codes (`NWP-ANCHOR-NOT-LEADER`, `NWP-ANCHOR-EPOCH-FENCED`, `NWP-BRIDGE-DIRECTION-UNSUPPORTED`, `NDP-CLUSTER-SPLIT`, `NIP-CERT-CAPABILITIES-EXCEEDED`) plus the profile-wave additions |

**Daemons** — ✅ `nps-registry` implements CR-0009 highest-epoch resolution + `NDP-CLUSTER-SPLIT`. Still open for later: `nps-ingress` full `TC-N2-*` / `TC-N2-HA-*` L2 vector coverage; `nps-runner` `SpawnSpec` OCI-image resolution + lease-renewal edge cases.

**Out of scope (→ beta.1)**: the NIP Phase-3 **flag day** itself (making enforcement MUST by default); multi-region NPS Cloud CA (Phase 3); QUIC production hardening beyond conformance vectors.

---

## alpha.18 — 2026-08-15 ✅ — **released to all registries** — **scope reduced**

> **Theme as planned**: *Pre-beta.1 Hardening* — production-harden every protocol and clear release-engineering debt ahead of the `v1.0.0-beta.1` NIP Phase-3 flag day, with NPS-Dev#90 (the NWP stateful LLM context/delta contract) as the single scoped design exception.
>
> **Theme as shipped**: the design exception became the release. alpha.18 delivered the **NPS-CR-0011 stateful LLM context contract on NWP 0.21**, the NIP 0.14 authorization surface it needs, and the release-engineering cleanup — but **three of the five per-protocol hardening tracks did not ship**. See *Scope reduction* below; they carry over to [alpha.19](#alpha19--next-target-2026-10--protocol-hardening-carry-over).

**Versions shipped**: NCP 0.11 (unchanged) / NWP **0.21** / NIP **0.14** / NDP 0.12 (unchanged) / NOP 0.9 (unchanged); `error-codes.md` **1.9** / `frame-registry.yaml` 0.14 (unchanged).

**Scope reduction — what did not ship.** The standing rule since alpha.11 is that every alpha advances all five protocols in **spec *and* all six SDKs** in lockstep. **alpha.18 is the first release since alpha.11 that does not meet it**, and the roadmap records that rather than restating the plan as if it had been met. NCP 0.12, NDP 0.13, NOP 0.10, and `frame-registry.yaml` 0.15 were planned for this release and were **not** started; the CHANGELOG never claimed them. The three tracks move to alpha.19 unchanged in substance.

**Release gates**:

1. **Spec content** — ✅ **partial (2 of 5 protocols)**. NWP v0.21 and NIP v0.14 advanced; `error-codes.md` v1.9. NCP / NDP / NOP unchanged — see *Scope reduction*.
2. **Cross-SDK implementation parity** (hard gate, applied to what shipped) — ✅ **done**. The CR-0011 stateful LLM context is implemented in all six SDKs (Go / Java / Python / Rust / TypeScript / .NET), each with passing suites: stateful NDJSON streaming with atomic terminal-frame commit, abnormal-termination abort, and idempotent replay of a completed sequence under a fresh `stream_id`.
3. **Conformance** — ✅ **done**. 19 shared CR-0011 vectors, made **fixture-driven** in all six SDKs — each vector's `input`, `pre_state`, and `expected` are executed and validated rather than the fixture serving as an ID dispatch list. Six-SDK black-box coverage for stateful reconnect, lost-response recovery, single-winner concurrent append/CAS, and process-local context loss after server restart.
4. **Release-engineering debt (P18-0)** — ✅ **done**. TypeScript `VERSION` now derives from package metadata instead of a hardcoded constant; the Go support floor was lowered from an unsupportable minimum to **Go 1.23**; Java MessagePack → 0.9.11, Jackson → 2.18.9, BouncyCastle → 1.84 under the OSV gate; standalone materialization deletes stale source while preserving distribution-only files; standalones receive the conformance fixtures they execute; registry preflight proves publish capability rather than anonymous read.
5. **Distribution** — ✅ **done**. Dev synchronized to `NPS-Release/spec`, all six standalone SDK repositories, daemon/tool distributions, and mirrors; `NPS-Release/version.yaml` bumped last; tagged and published to every registry (PyPI, npm, crates.io ×8, NuGet ×11 to nuget.org **and** the InnoLotus Nexus, Maven Central, Go proxy), plus GitHub releases and Gitee mirrors.
6. **CN translation** — ✅ No inherited backlog: the NPS-1..NPS-5 CN specifications reached structural and semantic parity with EN in NPS-Dev#87 before the release.
7. **Status hygiene** — ✅ **CR-0011** → Implemented; NPS-Dev#86 / #87 merged, #88 / #89 / #90 closed; the tracker carries no open issues at release.

**Per-protocol deliverables**:

| Protocol | Version | Notes |
|---|---|---|
| **NWP** | v0.21 | **NPS-CR-0011 stateful LLM context** — owner-bound opaque context IDs; `create` / `append` / `fork` / `reset` / `status` / `release`; compare-and-swap versions; atomic unary and async cancellation semantics; NWM 0.2 discovery; deterministic errors; shared lifecycle / replay / restart / accounting vectors. Stateless completion stays compatible and a stateful request never silently falls back. Also: official **LLM usage telemetry** (`input_tokens`, `output_tokens`, prefix/KV-cache hit, reused and evaluated tokens) and unary `CapsFrame.request_id` correlation (NPS-Dev#88), with `CapsFrame.cached` kept explicitly distinct from model prefix/KV-cache reuse; `LlmUsageDto.wire_input_bytes` for decoder-boundary request measurement |
| **NIP** | v0.14 | `llm:context` authorization for the CR-0011 surface. All six stateful LLM Action coordinators **fail closed** when no deployment authorizer is configured, and pass the exact admission/commit capability set (`llm:complete` + `llm:context`, plus stream/tool capabilities when used) to that authorizer |
| **NCP** | v0.11 *(unchanged)* | Planned 0.12 hardening — runtime keepalive timers, deterministic timeout closure, QUIC connection migration / 0-RTT rejection / flow control / backpressure, shared idle/dead-peer/oversized-Nop vectors — **not started; carried to alpha.19** |
| **NDP** | v0.12 *(unchanged)* | #89 (.NET nullable-`UInt64` NativeAOT resolver fix for `AnnounceFrame`) shipped as an implementation fix without a spec bump. Planned 0.13 hardening — sequence/epoch fence persistence and recovery, restart/partition/stale-entry/equal-epoch-split/loop fault vectors — **not started; carried to alpha.19** |
| **NOP** | v0.9 *(unchanged)* | Planned 0.10 hardening — bounded sliding-window replay/eviction, TTL expiry, `weighted_first_k`, `merge_all` executed identically across six runtimes, loss/reorder/duplicate/timeout vectors — **not started; carried to alpha.19** |
| **Shared** | error-codes v1.9 · frame-registry v0.14 *(unchanged)* | `NPS-LIMIT-RESOURCE` for bounded live-object limits, plus the CR-0011 deterministic error set. `frame-registry.yaml` 0.15 was planned alongside the three unshipped protocol bumps and moves with them |
| **Benchmark** | — | Strict-native CR-0011 second-turn benchmark using the official MessagePack `ActionFrame` decoder: a deterministic gate verifies delta-only role/tool semantic parity with fallback disabled, and separately reports lower decoder `wire_input_bytes` and runtime `evaluated_tokens` |

**Out of scope (→ beta.1)**: the NIP Phase-3 **flag day** itself; multi-region NPS Cloud CA (Phase 3); the 1.0 spec freeze.

---

## alpha.19 — 🚧 next (target 2026-10) — **Protocol Hardening (carry-over)**

> **Theme**: *Protocol Hardening* — finish the three per-protocol hardening tracks that alpha.18 planned but did not ship, and restore the standing lockstep rule before `v1.0.0-beta.1`.
>
> **Standing rule (since alpha.11)**: every alpha advances all five protocols in **spec *and* all six SDKs** in lockstep — no ".NET-reference-first" gaps. Each item below lands in spec + go/java/python/rust/typescript/.NET + conformance vectors + CN translation + the four doc surfaces. alpha.18 broke this rule; alpha.19 exists to restore it.

**Target versions**: NCP **0.12** / NDP **0.13** / NOP **0.10**; `frame-registry.yaml` **0.15**. NWP and NIP hold at 0.21 / 0.14 unless a hardening delta requires otherwise.

**Baseline (2026-08-16)**: alpha.18 is published to every registry; the tracker has no open issues; the CN specs are at parity with EN; the release-engineering debt listed under alpha.18 P18-0 is cleared.

**Per-protocol hardening** (carried from alpha.18 unchanged in substance):

| Protocol | Version | Existing baseline | Acceptance |
|---|---|---|---|
| **NCP** | 0.12 | NopFrame, `ping_interval_ms`, dead-peer rules, and native TCP/QUIC profiles already exist | Runtime keepalive timers and deterministic timeout closure in all six SDKs; QUIC connection migration, 0-RTT rejection, flow-control, and backpressure policy; shared idle/dead-peer/oversized-Nop vectors |
| **NDP** | 0.13 | Resolve-time staleness, health, three-hop federation, registry profile, and `cluster_epoch` already exist | Persist and recover sequence/epoch fences; prove restart, partition, stale-entry, equal-epoch split, loop, and recovery behavior with shared fault vectors |
| **NOP** | 0.10 | Aggregate strategies, ACK/NAK fields, result TTL, and portable orchestrator profile already exist | Execute bounded sliding-window replay/eviction, TTL expiry, `weighted_first_k`, and `merge_all` identically across six runtimes; add loss/reorder/duplicate/timeout conformance vectors |

**Also carried from the alpha.18 plan**: NWP resumable-subscription enforcement and portable stability/SLA/billing metadata; NIP short-lived certificate renewal interoperability, fail-closed OCSP/CRL behavior under timeout/stale/unknown responses, and the advisory tool that reports what beta.1 Phase-3 enforcement would reject. These were planned as NWP/NIP hardening alongside the version bumps that *did* ship for other reasons, so they are not covered by 0.21 / 0.14 as released.

**Execution gates**:
1. **P19-1 — Spec/design freeze**: write one normative hardening delta per protocol, bump NCP / NDP / NOP and `frame-registry.yaml`, and land EN/CN plus shared positive/negative/fault vectors together.
2. **P19-2 — Runtime parity**: implement every P19-1 behavior in all six SDKs. A field-only DTO port does not satisfy this gate; timers, persistence, cancellation, expiry, replay, and failure paths must execute.
3. **P19-3 — Fault and package gates**: run six-language suites, NativeAOT, race/concurrency tests where available, fault vectors, package dry-runs, and security/dependency scans at the documented minimum toolchains.
4. **P19-4 — Distribution**: materialize standalones with deletion and distribution excludes, vendor their conformance fixtures, reach zero unexplained Dev→Release/SDK drift, then perform the normal pre-release review. Tagging and publishing remain separately approved actions.

**Release-runbook invariants**: crates is **8** crates including `nps-conformance`; the NuGet family is **11** packages; standalone sync deletes stale source while preserving documented distribution-only files; every standalone receives the conformance fixtures it executes; Maven packaging has a Python `zipfile` fallback; registry preflight proves publish capability (`cargo owner --list`, npm granular read/write token with publish 2FA bypass), not merely anonymous read access.

**Out of scope (→ beta.1)**: the NIP Phase-3 **flag day** itself; multi-region NPS Cloud CA (Phase 3); the 1.0 spec freeze.

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
- [x] Tier-3 BinaryVector v1 specification (CR-0008)
- [ ] Tier-3 MatrixTensor / additional dtype extensions

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
