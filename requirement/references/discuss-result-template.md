# discuss-result.md Template

Full format for the technical decisions knowledge base generated at Stage 3.

---

```markdown
---
generated_at: YYYY-MM-DDTHH:MM:SSZ
git_sha: <sha>
feature: <one-line feature description>
discuss_ref: design/<date>/discuss.md
---

# Technical Decisions: <Feature Name>

> **Purpose**: This document is the single source of truth for all subagents implementing this feature.
> Reading ARCH.md + feat.md + this document should be sufficient to implement, review, and test.

---

## Summary of Decisions

| ID | Decision | Chosen Option | Key Constraint |
|---|---|---|---|
| D1 | [title] | Option X — [name] | [one-line constraint for future reference] |
| D2 | [title] | Option Y | |
| ... | | | |

---

## §D1 · <Decision Title>

**Chosen**: Option X — <name>

**Rationale**: [2–4 sentences explaining why. Reference the ARCH.md invariant or data model element that this aligns with, or the product constraint that drove the choice.]

### Implementation Notes

[Concrete details a dev-member needs to implement this decision. Write at skeleton depth — enough to make architectural choices unambiguous, not full implementation.]

**DB schema** (if applicable):
```sql
CREATE TABLE example (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id),
  name TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX ON example (user_id);
```

**Key types / interfaces** (if applicable):
```typescript
interface ExampleRecord {
  id: string
  userId: string
  name: string
  createdAt: Date
}

// Service layer
function createExample(userId: string, name: string): Promise<ExampleRecord>
function getExample(id: string, userId: string): Promise<ExampleRecord | null>
```

**Key algorithm or flow** (if applicable):
```
POST /api/examples
  → auth middleware (existing, INV-1)
  → ExampleRoute.create()
    → validate: name required, max 200 chars
    → ExampleService.create(userId, name)
      → ExampleRepository.insert(...)
    ← returns ExampleRecord
  ← 201 { id, name, createdAt }
```

### Constraints for dev-member

- [Explicit constraint the dev-member must not violate — e.g., "Must go through ExampleRepository, never direct DB access (INV-2)"]
- [Another constraint — e.g., "Migration must be additive — do not drop or rename existing columns"]

### Tech Debt Introduced

- **TD-X**: [description] — [why deferred, what triggers cleanup]

---

## §D2 · <Decision Title>

**Chosen**: Option Y — <name>

**Rationale**: [explanation]

### Implementation Notes

[skeleton depth notes]

### Constraints for dev-member

- [constraint]

### Tech Debt Introduced

- **TD-X**: [description]

---

[Repeat §Dn for each decision]

---

## §Integration Notes

[Optional section for cross-cutting concerns that span multiple decisions]

**Request flow across all decisions**:
```
[end-to-end flow showing how D1, D3, D5 interact at runtime]
```

**Migration order**:
1. [step 1 — must come first because ...]
2. [step 2]
3. [step 3]

**Env variables required**:
```
NEW_VAR=...        # used by §D3 — [what for]
ANOTHER_VAR=...    # used by §D5
```

---

## §Tech Debt Register

All TD items introduced by this feature. Dev-member must link each item to the relevant ARCH.md §9 entry after implementation.

| ID | Description | Severity | Introduced by | Trigger for cleanup |
|---|---|---|---|---|
| TD-1 | [description] | high / med / low | D2 | [what event should trigger fixing this] |
| TD-2 | [description] | med | D4 | [trigger] |

---

## §ARCH.md Update Triggers

After implementation, check whether these sections need updating:

- [ ] §2 Quick Reference — any new key files?
- [ ] §4 Module Boundaries — any new or changed module?
- [ ] §5 Data Model — new tables or columns?
- [ ] §6 Typical Data Flows — new flows?
- [ ] §7 Key Invariants — any new invariant added or existing one changed?
- [ ] §8 Terminology — any new terms introduced?
- [ ] §9 Tech Debt — TD items from this feature

If any box is checked, ARCH.md update is mandatory in the same PR.
```

---

## Writing Notes

### Skeleton Depth

The goal is to make every architectural decision unambiguous without over-specifying implementation details.

**Right depth** — sufficient for a dev-member to know what to build:
- Table schema (columns, types, constraints, indexes)
- Function/method signatures (name, parameters, return type)
- HTTP contract (route, method, request shape, response shape, key error codes)
- Key algorithm described in pseudocode or a flow diagram

**Too shallow** — leaves architectural decisions open:
- "Add a service that handles X" (no interface)
- "Store data in the DB" (no schema)
- "The frontend will call an API" (no contract)

**Too deep** — dev-member's job, not yours:
- Full implementation code
- Error handling for every edge case
- Test code

### Self-Containment Requirement

A dev-member who has never been in this conversation must be able to read:
1. ARCH.md (for project context)
2. feat.md (for existing features)
3. This discuss-result.md (for this feature's decisions)

...and implement without asking any clarifying questions. If there's a gap, fill it.

### Code Skeletons

Write skeletons in the project's actual language and framework. Read `src/index.ts` or `package.json` to confirm — don't guess.

For SQL, use the exact dialect (PostgreSQL-specific syntax if the project uses Postgres, not generic SQL).

For TypeScript, match the naming conventions in ARCH.md §8 (Terminology). If a term used in discuss-result.md doesn't appear in ARCH.md §8, add it to the §ARCH.md Update Triggers list.
