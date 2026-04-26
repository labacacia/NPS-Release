English | [中文版](./template.cn.md)

---
**RFC Number**: NPS-RFC-NNNN
**Title**: _one-line summary, imperative mood — "Add tensor encoding tier"_
**Status**: Draft
**Author(s)**: _Name <email> (role / org)_
**Shepherd**: _TBD — assigned on PR open_
**Created**: YYYY-MM-DD
**Last-Updated**: YYYY-MM-DD
**Accepted**: _(set on merge to `dev`)_
**Activated**: _(set when first reference SDK ships)_
**Supersedes**: _— RFC number(s) this replaces, if any_
**Superseded-By**: _— RFC number that replaces this, set when superseded_
**Affected Specs**: _— e.g. NPS-1 NCP, frame-registry.yaml, error-codes.md_
**Affected SDKs**: _— `.NET, Python, TypeScript, Java, Rust, Go` (cross out those not affected)_
---

# NPS-RFC-NNNN: {Title}

## 1. Summary

_Two or three sentences. A reader must be able to decide "is this
relevant to me?" from this paragraph alone. State **what** changes and
**who** is affected — not yet **why**._

## 2. Motivation

_What problem does this solve? Why now? What's the cost of not doing
it? Use concrete data where possible — measured latency, measured size,
a real incident, a real user request._

_If this is responding to external pressure (a spec from another body,
a registry policy, a security disclosure), link to it._

## 3. Non-Goals

_Explicit scope limits. "This RFC does **not** address X" lines save
weeks of review-cycle drift. If a related concern is out of scope, say
so here and — if appropriate — open a stub issue for it._

## 4. Detailed Design

### 4.1 Wire Format / Frame Changes

_Bit-exact layout. Field table. Examples of encoded bytes. If this
changes `frame-registry.yaml`, show the diff inline._

```yaml
# Example frame-registry.yaml delta
- id: 0xNN
  name: NewFrame
  ...
```

### 4.2 Manifest / NWM Changes

_Any `/.nwm` schema additions. New fields MUST declare required /
optional and default behavior for agents that don't understand them._

### 4.3 Error Codes

_New entries in `spec/error-codes.md`. Format: `{PROTOCOL}-{CATEGORY}-{DETAIL}`.
Each one MUST map to an NPS status code from `spec/status-codes.md`;
if a new status code is needed, declare it here and update
`status-codes.md` in the same PR._

### 4.4 State Machines / Flows

_Sequence diagram or ASCII flow when the change affects protocol
timing. Include timeout values, retry behavior, and what counts as a
successful completion._

### 4.5 Backward Compatibility

_Answer each in one line — "yes / no / n/a" + rationale:_

- Will old Agents understand the new frames? _(no → breaking)_
- Will old Nodes ignore the new fields safely? _(depends on field
  required vs optional)_
- What does a version-mismatch error look like on the wire?
- Does `min_agent_version` need to be raised? _(if yes → 21-day
  window)_

---

## 5. Alternatives Considered

_At least two. "Do nothing" is always one alternative — spell out
what **that** costs. For each alternative, list the tradeoff that
made it lose._

### 5.1 _Alternative name_

_Description. Why rejected._

### 5.2 Do nothing

_What breaks if we don't act? What's the ceiling on how long we can
defer?_

---

## 6. Drawbacks & Risks

_What could go wrong even if this RFC is correct?_

- Migration cost: _estimate in SDK-weeks_
- Attack surface: _new parser paths? new signature domains?_
- Ecosystem fragmentation: _are there outside-project consumers who
  depend on the old behavior?_
- Reversibility: _if this turns out to be a bad idea, what does
  rolling it back cost?_

---

## 7. Security Considerations

_Mandatory section. Even "none" must be stated explicitly._

- New attack surface? _New parsers = new CVE class._
- New cryptographic assumptions?
- Impact on existing signature / encryption / replay protections?
- DoS vectors: _is the new frame bounded in size / processing cost?_

---

## 8. Implementation Plan

### 8.1 Phasing

_Break into at most 4 phases. Each phase must be individually
shippable and individually reversible._

| Phase | Scope | Exit criterion |
|-------|-------|----------------|
| 1 | Spec merged + reference SDK behind feature flag | Unit tests green; one interop test between two SDKs |
| 2 | All 6 SDKs implement; flag still default-off | Cross-SDK interop matrix green |
| 3 | Flag flips to default-on in `NPS-RELEASE-NOTES.md` | No regressions in benchmark suites for 1 release cycle |
| 4 | Remove flag; behavior mandatory | Deprecated fallback codepath removed |

### 8.2 SDK Coverage Matrix

Each SDK owner signs off or declares a follow-up:

| SDK | Owner | Status | Notes |
|-----|-------|--------|-------|
| .NET | | | |
| Python | | | |
| TypeScript | | | |
| Java | | | |
| Rust | | | |
| Go | | | |

### 8.3 Test Plan

_What new tests land with this RFC? Which existing tests change?
What's the cross-SDK interop test?_

### 8.4 Benchmarks

_If this affects wire size or token cost, a benchmark MUST be added
to `impl/dotnet/benchmarks/` (or equivalent) before Phase 2 exit.
State the threshold that would trigger a re-evaluation of the RFC._

---

## 9. Empirical Data

_Numbers go here. Prototypes, microbenchmarks, measurements from an
experimental branch. If there aren't any yet, write "none yet" and
commit to having some before Accepted state. **"It will probably be
faster"** is not data._

| Metric | Baseline | Proposed | Delta | Method |
|--------|----------|----------|-------|--------|
| | | | | |

---

## 10. Open Questions

_Track unresolved issues. Each entry MUST have either a resolution or
an explicit defer-with-tracking-issue before the RFC moves to Accepted._

- [ ] _Question 1_ — {owner} — {target resolution date}
- [ ] _Question 2_

---

## 11. Future Work

_What this RFC deliberately leaves for later. Link to follow-up RFCs
when they open._

---

## 12. References

- _Related RFCs in this repo_
- _External specs / papers / prior art_
- _Discussion threads (GitHub issues, Discussions)_

---

## Appendix A. Revision History

| Date | Author | Change |
|------|--------|--------|
| YYYY-MM-DD | | Initial draft |
