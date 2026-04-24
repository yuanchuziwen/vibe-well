# discuss.md Template

Full format for the decision-point document generated at Stage 2.

---

```markdown
---
generated_at: YYYY-MM-DDTHH:MM:SSZ
git_sha: <sha>
feature: <one-line feature description>
status: draft | decisions-pending | resolved
---

# Requirement Discussion: <Feature Name>

> **Background**: [1–2 sentences — what the user asked for, what problem it solves]
> **Scope**: [what this doc covers and does not cover]

---

## Decision Overview

| ID | Decision | Status | Chosen |
|---|---|---|---|
| D1 | [title] | ⬜ pending / ✅ resolved | — |
| D2 | [title] | ⬜ pending | — |
| ...| | | |

---

## D1 · <Decision Title>

> **Why this matters**: [one sentence — what breaks or gets harder if we get this wrong]

### Options

**Option A — <name>**
- [what it is, concretely]
- Pros: [2–3 bullet points]
- Cons: [2–3 bullet points]
- Fits existing invariants: ✅ / ⚠️ / ❌ [explain if not ✅]

**Option B — <name>**
- [what it is, concretely]
- Pros: [2–3 bullet points]
- Cons: [2–3 bullet points]
- Fits existing invariants: ✅ / ⚠️ / ❌

**Option C — <name>** *(if applicable)*
- ...

### Recommendation

**Option X** — [reason in 2–3 sentences. Reference ARCH.md invariants or data model if relevant.]

### Cold Water ⚠️

[What could go wrong with the recommended option? Migration risk, concurrency edge case, hidden complexity, future regret. Be specific. This is where you surface the non-obvious.]

### Decision

> [User fills in after reading: "Go with Option X" or "Go with Option A but modified: ..."]

---

## D2 · <Decision Title>

[same structure]

---

## D3 · <Decision Title>

[same structure]

---

## Total Scope Estimate

| Phase cluster | Decisions covered | Estimated size |
|---|---|---|
| [name] | D1, D2 | S / M / L / XL |
| [name] | D3, D4, D5 | M |

**Size key**: S ≈ 0.5–1 day, M ≈ 1–2.5 days, L ≈ 3–5 days, XL ≈ 1+ week (full team)

**Total**: ~ [range]

---

## Out of Scope (Future Considerations)

- [item] — deferred because [reason]
- [item]
```

---

## Writing Notes

### Dn Selection Criteria

Include a decision point when:
- The choice affects architecture (schema, module boundary, invariant)
- Multiple reasonable options exist with meaningful tradeoffs
- The user needs to weigh product vs technical priorities
- Getting it wrong means painful rework later

Do NOT include a decision point when:
- Industry best practice is clear and ARCH.md invariants are satisfied
- It's a pure implementation detail (naming, file organization)
- One option is obviously better with no real tradeoff

### Cold Water Examples

Good cold water notes address:
- **Migration safety**: "If we choose Option B, the existing `documents.content` column will need a backfill migration on a potentially large table — must be tested under concurrent writes."
- **Invariant conflict**: "Option A bypasses the repository pattern (INV-2) — services would access DB directly. Requires an explicit decision to make an exception."
- **Hidden coupling**: "Option C shares the auth token with the external service, which means a token rotation on our side immediately breaks the external integration."
- **Future regret**: "Option A is simpler now, but locks us into a single-tenant model. When multi-tenant comes up (and it will), the migration will be a rewrite."

### Dependency Ordering

Order Dn so that decisions that unlock other decisions come first. For example:
- D1: storage backend → affects D2 (schema design), D3 (query patterns), D5 (migration strategy)
- D2: schema design → only makes sense after D1 is resolved
- D4: auth approach → independent of D1–D3, can be placed anywhere

If there are independent decision tracks, group them:
```
Track A (storage):  D1 → D2 → D3
Track B (auth):     D4 → D5
Track C (UI):       D6 → D7
```
