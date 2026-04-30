English | [中文版](./README.cn.md)

# NPS Change Requests (CRs)

A **Change Request (CR)** is a lightweight design artifact used during the
**pre-1.0** phase of the NPS suite to record the intent, motivation, and
shape of a specification or implementation change before (or alongside)
the code that lands it.

## CR vs RFC

| Aspect | CR (this directory) | RFC ([../rfcs/](../rfcs/)) |
|---|---|---|
| Phase | Pre-1.0 (alpha / beta) | Post-1.0 stable |
| Gating | None — implementation may proceed without external acceptance | Required — Accepted on `dev` PR before any implementation |
| Reviewers | Author + (optionally) one independent reviewer | Shepherd + community discussion window |
| Numbering | `NPS-CR-NNNN` | `NPS-RFC-NNNN` |
| Lifecycle | Draft → Implemented (or Withdrawn) | Draft → Proposed → Accepted → Activated → (Superseded) |
| Status retention | Kept as historical record after implementation | Permanent record; supersession via new RFC |
| Scope | Single coherent change, often spanning spec + impl + tests | Same |

CRs exist because, while the suite is still 0.x and has no production
deployments to protect, the friction of the full RFC process (shepherd
assignment, formal discussion window, accepted-on-merge ceremony) buys
little. Once the suite reaches v1.0, all spec-affecting changes
**MUST** instead be drafted as RFCs under [../rfcs/](../rfcs/) and
follow the process documented there.

## How a CR is structured

A CR document covers, at minimum:

1. **Summary** — one paragraph stating what changes.
2. **Motivation** — why this change is needed now.
3. **Specification changes** — exact text or diff for `spec/` files.
4. **SDK changes** — language-by-language impact.
5. **Conformance changes** — added or modified test items.
6. **Migration impact** — internal and external users.
7. **Out of scope** — explicit non-changes to prevent scope creep.
8. **Acceptance criteria** — concrete completion checklist.
9. **CHANGELOG entry** — proposed text.
10. **Open questions** — items the author wants reviewers to weigh in on.

The structure mirrors a formal RFC closely enough that a CR can be
promoted to an RFC verbatim if it ever needs to be (e.g. if a CR drafted
late in 0.x is still pending when v1.0 is cut).

## Index

| CR | Title | Target | Status |
|---|---|---|---|
| [NPS-CR-0001](./NPS-CR-0001-anchor-bridge-split.md) | Split Gateway Node into Anchor Node + Bridge Node | v1.0-alpha.3 | Implemented (2026-04-26) |
| [NPS-CR-0002](./NPS-CR-0002-anchor-topology-queries.md) | Standard topology query types for Anchor Node (`topology.snapshot` / `topology.stream`) | v1.0-alpha.4 | Implemented (2026-04-27) |

## Authoring a new CR

1. Pick the next available `NNNN` from the index above.
2. Copy the structure of an existing CR (e.g. CR-0001) — there is no
   formal template file; the structure is intentionally light.
3. Write English (`NPS-CR-NNNN-<slug>.md`). A Chinese counterpart is
   **not required** for CRs — they are informal pre-1.0 planning artifacts.
   The downstream `spec/*.md` text that lands the change will carry the
   standard English-primary + Chinese-secondary pair per project convention.
4. Add the row to the Index table above.
5. Open a `feat/cr-NNNN-<slug>` branch and start implementing.
