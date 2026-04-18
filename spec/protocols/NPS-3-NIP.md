English | [中文版](./NPS-3-NIP.cn.md)

# NPS-3: Neural Identity Protocol (NIP)

**Spec Number**: NPS-3  
**Status**: Draft  
**Version**: 0.2  
**Date**: 2026-04-12  
**Port**: 17433 (default, shared) / 17435 (optional dedicated)  
**Authors**: Ori Lynn / INNO LOTUS PTY LTD  
**Depends-On**: NPS-1 (NCP v0.3)  

---

## 1. Terminology

The key words "MUST", "MUST NOT", "SHOULD", and "MAY" in this document are to be interpreted as described in RFC 2119.

---

## 2. Protocol Overview

NIP issues a verifiable Neural Identity (NID) to every AI agent, NWP node, and human operator, carrying capability declarations and scope-bound permissions. It supports trust-chain propagation and real-time revocation. NIP is the heart of the NPS security model.

### 2.1 Actors

| Role | Description |
|------|-------------|
| Root CA | Root certificate authority; stored offline |
| Org CA | Organization intermediate CA; one per organization |
| Agent | AI agent holding an NID certificate |
| Node | NWP node holding an NID certificate |
| Operator | Human administrator holding an Operator certificate |
| Verifier | The party verifying an NID (usually a Node) |

### 2.2 CA Hierarchy

```
Root CA (offline, rarely used)
    │
    └── Org Intermediate CA (one per org, 1-year validity)
            ├── Agent Certificate    (30-day validity, auto-renewal supported)
            ├── Node Certificate     (90-day validity)
            └── Operator Certificate (1-year validity, bound to MFA)
```

---

## 3. NID Format

```abnf
nid           = "urn:nps:" entity-type ":" issuer-domain ":" identifier
entity-type   = "agent" / "node" / "org"
issuer-domain = <RFC 1034 domain>
identifier    = 1*(ALPHA / DIGIT / "-" / "_" / ".")
```

**Examples**

```
urn:nps:agent:ca.innolotus.com:550e8400-e29b-41d4    ← AI agent
urn:nps:node:api.myapp.com:products                   ← NWP node
urn:nps:org:mycompany.com                              ← Organization CA
```

---

## 4. Signature Algorithms

| Algorithm | Use | Key length |
|-----------|-----|-----------|
| **Ed25519** (primary) | High-frequency agent verification | 32-byte private key / 32-byte public key |
| **ECDSA P-256** (fallback) | Compatibility scenarios | 32-byte private key / 64-byte public key |

Public-key encoding: `{algorithm}:{base64url(DER)}`, e.g. `ed25519:MCowBQYDK2VwAyEA...`.

---

## 5. Frame Types

### 5.1 IdentFrame (0x20)

Agent identity declaration carrying the certificate. Sent as the handshake frame on every new connection.

**Field definitions**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `frame` | uint8 | required | Fixed value `0x20` |
| `nid` | string | required | Agent NID |
| `pub_key` | string | required | Public key; format `{alg}:{base64url}` |
| `capabilities` | array | required | Capabilities held by the agent |
| `scope` | object | required | Access-scope declaration |
| `issued_by` | string | required | Issuer NID (Org CA) |
| `issued_at` | string | required | Issue time (ISO 8601 UTC) |
| `expires_at` | string | required | Expiration time (ISO 8601 UTC) |
| `serial` | string | required | Certificate serial (globally unique per Org CA, hex) |
| `signature` | string | required | CA's signature over this frame; format `{alg}:{base64url}` |
| `metadata` | object | optional | Agent metadata — see below |

**`metadata` fields (optional)**

| Field | Type | Description |
|-------|------|-------------|
| `model_family` | string | Model-family identifier used by the agent, e.g. `"openai/gpt-4o"`, `"anthropic/claude-4"` |
| `tokenizer` | string | Tokenizer identifier used by the agent, e.g. `"cl100k_base"` |
| `runtime` | string | Agent runtime identifier, e.g. `"langchain/0.2"`, `"autogen/0.4"` |

The `metadata` field is not included in the signature computation; an agent MAY set it at runtime. Nodes use the `tokenizer` in `metadata` to perform NPT auto-matching (see [token-budget.md](../token-budget.md)).

**Standard `capabilities` values**

| Capability | Description |
|------------|-------------|
| `nwp:query` | May query Memory Nodes |
| `nwp:action` | May invoke Action Nodes |
| `nwp:stream` | May receive StreamFrame responses |
| `ncp:stream` | May initiate NCP streaming |
| `nop:delegate` | May delegate subtasks to other agents |
| `nop:orchestrate` | May act as an orchestrator and emit TaskFrames |

**`scope` field**

```json
{
  "nodes":            ["nwp://api.myapp.com/*"],
  "actions":          ["orders:read", "orders:create"],
  "max_token_budget": 50000
}
```

**Signature computation**

The signature is computed over the canonical JSON of the IdentFrame with the `signature` field removed (keys sorted alphabetically, no whitespace).

**Full example**

```json
{
  "frame": "0x20",
  "nid": "urn:nps:agent:ca.innolotus.com:550e8400-e29b-41d4",
  "pub_key": "ed25519:MCowBQYDK2VwAyEA...",
  "capabilities": ["nwp:query", "nwp:action", "ncp:stream"],
  "scope": {
    "nodes":            ["nwp://api.myapp.com/*"],
    "actions":          ["orders:read", "orders:create"],
    "max_token_budget": 50000
  },
  "issued_by":  "urn:nps:org:mycompany.com",
  "issued_at":  "2026-04-10T00:00:00Z",
  "expires_at": "2026-05-10T00:00:00Z",
  "serial":     "0x0A3F9C",
  "signature":  "ed25519:3045022100..."
}
```

---

### 5.2 TrustFrame (0x21)

Cross-CA trust-chain propagation and capability grant (commercial feature, NPS Cloud).

```json
{
  "frame":       "0x21",
  "grantor_nid": "urn:nps:org:org-a.com",
  "grantee_ca":  "urn:nps:org:org-b.com",
  "trust_scope": ["nwp:query"],
  "nodes":       ["nwp://api.org-a.com/public/*"],
  "expires_at":  "2026-12-31T00:00:00Z",
  "signature":   "ed25519:..."
}
```

---

### 5.3 RevokeFrame (0x22)

Revokes an NID or a specific capability.

```json
{
  "frame":      "0x22",
  "target_nid": "urn:nps:agent:ca.innolotus.com:550e8400-e29b-41d4",
  "serial":     "0x0A3F9C",
  "reason":     "key_compromise",
  "revoked_at": "2026-04-10T12:00:00Z",
  "signature":  "ed25519:..."
}
```

`reason` values: `key_compromise` / `ca_compromise` / `affiliation_changed` / `superseded` / `cessation_of_operation`.

---

## 6. Certificate Lifecycle

```
Registration     Auto-renewal           Revocation        CA rotation
     │                 │                     │                │
     ↓                 ↓                     ↓                ↓
CA issues         7 days pre-expiry     RevokeFrame      Old + new CA
IdentFrame        Agent triggers         takes effect     in parallel
30-day validity   CA issues new cert     immediately       for 30 days
                  Both valid 1 hour      Nodes reject      Smooth rollover
                                         all requests
                                         for that NID
```

---

## 7. Verification Flow

```
Node receives IdentFrame
  │
  ├─ 1. Check expires_at > now  (expired → NIP-CERT-EXPIRED)
  ├─ 2. Check issued_by is in NWM trusted_issuers  (not trusted → NIP-CERT-UNTRUSTED-ISSUER)
  ├─ 3. Verify signature with issued_by CA public key  (fail → NIP-CERT-SIGNATURE-INVALID)
  ├─ 4. OCSP lookup (when NWM configures ocsp_url) or local CRL check  (revoked → NIP-CERT-REVOKED)
  ├─ 5. Check capabilities contains what the node requires  (missing → NIP-CERT-CAPABILITY-MISSING)
  └─ 6. Check scope.nodes covers the target node path  (not covered → NWP-AUTH-NID-SCOPE-VIOLATION)
        All checks pass → authorize the request
```

---

## 8. NIP CA Server OSS API

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/v1/agents/register` | Operator Cert | Register an agent; returns NID + IdentFrame |
| POST | `/v1/agents/{nid}/renew` | Agent Cert | Renew a certificate (up to 7 days before expiry) |
| POST | `/v1/agents/{nid}/revoke` | Operator Cert | Revoke an NID |
| GET | `/v1/agents/{nid}/verify` | None | Verify NID validity (OCSP lookup) |
| POST | `/v1/nodes/register` | Operator Cert | Register an NWP node; issues a Node Certificate |
| GET | `/v1/ca/cert` | None | CA public-key certificate |
| GET | `/v1/crl` | None | Certificate revocation list |
| GET | `/.well-known/nps-ca` | None | CA discovery endpoint |

**`/.well-known/nps-ca` response**

```json
{
  "nps_ca": "0.1",
  "issuer": "urn:nps:org:ca.mycompany.com",
  "display_name": "MyCompany NPS CA",
  "public_key": "ed25519:MCowBQYDK2VwAyEA...",
  "algorithms": ["ed25519", "ecdsa-p256"],
  "endpoints": {
    "register": "https://ca.mycompany.com/v1/agents/register",
    "verify":   "https://ca.mycompany.com/v1/agents/{nid}/verify",
    "ocsp":     "https://ca.mycompany.com/ocsp",
    "crl":      "https://ca.mycompany.com/v1/crl"
  },
  "capabilities": ["agent", "node", "operator"],
  "max_cert_validity_days": 30
}
```

---

## 9. Error Codes

| Error Code | NPS Status | Description |
|------------|------------|-------------|
| `NIP-CERT-EXPIRED` | `NPS-AUTH-UNAUTHENTICATED` | Certificate has expired |
| `NIP-CERT-REVOKED` | `NPS-AUTH-UNAUTHENTICATED` | Certificate has been revoked |
| `NIP-CERT-SIGNATURE-INVALID` | `NPS-AUTH-UNAUTHENTICATED` | Certificate signature verification failed |
| `NIP-CERT-UNTRUSTED-ISSUER` | `NPS-AUTH-UNAUTHENTICATED` | Issuer not in the trust list |
| `NIP-CERT-CAPABILITY-MISSING` | `NPS-AUTH-FORBIDDEN` | Certificate missing a required capability |
| `NIP-CERT-SCOPE-VIOLATION` | `NPS-AUTH-FORBIDDEN` | Certificate scope does not cover the target path |
| `NIP-CA-NID-NOT-FOUND` | `NPS-CLIENT-NOT-FOUND` | NID does not exist |
| `NIP-CA-NID-ALREADY-EXISTS` | `NPS-CLIENT-CONFLICT` | NID already exists (duplicate registration) |
| `NIP-CA-SERIAL-DUPLICATE` | `NPS-CLIENT-CONFLICT` | Certificate serial already in use |
| `NIP-CA-RENEWAL-TOO-EARLY` | `NPS-CLIENT-BAD-PARAM` | Renewal window not yet open |
| `NIP-CA-SCOPE-EXPANSION-DENIED` | `NPS-AUTH-FORBIDDEN` | Requested scope exceeds the parent scope |
| `NIP-OCSP-UNAVAILABLE` | `NPS-SERVER-UNAVAILABLE` | OCSP service temporarily unavailable |
| `NIP-TRUST-FRAME-INVALID` | `NPS-CLIENT-BAD-FRAME` | TrustFrame signature or format is invalid |

HTTP-mode status mapping: see [status-codes.md](../status-codes.md).

---

## 10. Security Considerations

### 10.1 Key storage
CA private keys MUST be stored in an HSM or in an encrypted key file (AES-256-GCM). The OSS reference implementation uses encrypted files and reserves an HSM interface.

### 10.2 Timing-attack mitigation
OCSP response times SHOULD be normalized (fixed 200 ms delay) to prevent certificate-status inference via response-time side channels.

### 10.3 No-scope-expansion principle
At every link in the delegation chain, scope MUST NOT exceed that of its parent. CAs MUST enforce this at issuance time.

---

## 11. Change Log

| Version | Date | Changes |
|---------|------|---------|
| 0.2 | 2026-04-12 | Unified port 17433; added `metadata` field to IdentFrame (tokenizer auto-match); error codes switched to NPS-status-code mapping; error-code list completed |
| 0.1 | 2026-04-10 | Initial spec: NID format, IdentFrame/TrustFrame/RevokeFrame, CA Server API, verification flow |

---

*Copyright: LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
