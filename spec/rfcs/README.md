English | [中文版](./README.cn.md)

# NPS RFC Process

This directory holds **Requests for Comments (RFCs)** — the formal record
of how a non-trivial change to the NPS suite was proposed, debated, and
decided. An RFC is **a thinking document, not an implementation plan**:
it exists so the argument survives the decision.

---

## When an RFC is required

An RFC **MUST** be opened before any PR that:

- Assigns a Reserved bit / code / namespace (e.g. encoding tier `0x02`,
  a new frame type, a new NPS status code range)
- Adds or removes a frame in `spec/frame-registry.yaml`
- Introduces a new node type or endpoint sub-path
- Changes on-the-wire semantics of an existing frame (breaking or silent)
- Defines a new authentication method or key algorithm
- Adds a new protocol layer to the suite (`NPS-6-*`)
- Raises the minimum Agent version (`min_agent_version` bump)

An RFC **SHOULD** be opened (but may be skipped with shepherd consent) for:

- Large cross-SDK refactors
- New compatibility bridges (e.g. a third-party protocol adapter)
- Policy changes: versioning, deprecation windows, release cadence

An RFC is **NOT** required for:

- Bug fixes and spec clarifications (typos, ambiguity resolution)
- Non-breaking SDK additions (new convenience helpers, new optional
  fields in already-extensible structs)
- Documentation, benchmarks, examples, CI changes
- Tool implementations (e.g. a new NIP CA Server backend)

> **Rule of thumb:** if landing the PR would force every SDK to either
> adopt or explicitly reject the change, it needs an RFC.

---

## Lifecycle

```
  Draft ─────► Accepted ─────► Active ─────► Superseded
   │              │
   │              └──► Withdrawn   (author pulls it)
   └──► Rejected
```

| State | Meaning | How to change |
|-------|---------|---------------|
| `Draft` | Author is writing / revising; PR open against `dev` | Author pushes commits |
| `Accepted` | Merged to `dev`; implementation may begin across SDKs | Shepherd merges PR after approval threshold met |
| `Active` | Implementation landed in ≥1 reference SDK; frame/spec changes integrated | Follow-up PR flips header; requires ≥1 SDK + green CI |
| `Superseded` | A later RFC replaces this one | The replacing RFC sets `Supersedes:` and this one is edited to set `Superseded-By:` |
| `Withdrawn` | Author no longer pursues it; preserved as history | Author edits header |
| `Rejected` | Discussion reached "no" consensus; preserved as history | Shepherd edits header with summary of why |

Rejected and Withdrawn RFCs are **never deleted** — the argument is the
deliverable.

---

## Approval threshold

Acceptance requires, at minimum:

- **1 shepherd** (maintainer, not the author) explicitly approves the PR
- **≥ 7 calendar days** of public discussion window (longer for breaking
  changes — see below)
- **Open questions** resolved or explicitly deferred with tracking issues
- **SDK-coverage matrix** filled: each SDK owner either signs off or
  declares a non-blocking follow-up

Extended windows:

| Change type | Minimum discussion window |
|-------------|---------------------------|
| New frame / new Reserved bit assignment | 14 days |
| Breaking wire-format change | 21 days |
| Raising `min_agent_version` | 21 days + one month deprecation notice |

---

## Numbering & filenames

- `NPS-RFC-{NNNN}-{kebab-slug}.md`  (English primary)
- `NPS-RFC-{NNNN}-{kebab-slug}.cn.md`  (Chinese mirror)

`NNNN` is a zero-padded four-digit number assigned **on PR open**,
monotonically increasing. Never reuse a number, even for Withdrawn /
Rejected RFCs.

---

## How to open an RFC

1. Copy `template.md` → `NPS-RFC-{next}-{slug}.md`
2. Copy `template.cn.md` → `NPS-RFC-{next}-{slug}.cn.md` and translate
3. Fill in the front matter (`Status: Draft`)
4. Open PR against `dev`, title format:
   `rfc: NPS-RFC-{NNNN} — {one-line summary}`
5. Tag at least one shepherd for review
6. Iterate on the PR; squash-merge is fine, commit history inside the
   RFC itself is what matters

Once merged to `dev` with `Status: Accepted`, implementation work can
start in any SDK. The RFC's `Status:` is updated to `Active` in a
follow-up PR once at least one reference SDK ships the change.

---

## Index

| Number | Title | Status | Accepted | Supersedes |
|--------|-------|--------|----------|------------|
| [0001](./NPS-RFC-0001-ncp-connection-preamble.md) | Add NCP connection preamble for native-mode traffic identification | Draft | _—_ | _—_ |
| [0002](./NPS-RFC-0002-x509-acme-nid-certs.md) | Adopt X.509 + ACME for NID certificates | Draft | _—_ | _—_ |
| [0003](./NPS-RFC-0003-agent-identity-assurance-levels.md) | Three-tier Agent identity assurance levels for anti-scraping / trust gating | Draft | _—_ | _—_ |
| [0004](./NPS-RFC-0004-nid-reputation-log.md) | Append-only NID reputation log (Certificate Transparency for Agents) | Draft | _—_ | _—_ |

---

## References

- Rust RFC process — https://github.com/rust-lang/rfcs
- Python PEPs — https://peps.python.org/
- Kubernetes KEPs — https://github.com/kubernetes/enhancements
- IETF RFC 7282 ("On Consensus and Humming in the IETF")
