---
name: turystack-modeling
description: "How a Turystack product is modelled — what becomes a table, how tables relate, how every row is scoped to a customer, and how a model changes without breaking the repositories already running on it. Open it before writing a migration, adding an entity, changing a cardinality, or designing a new domain; and open the canonical model file when the domain is one Turystack already ships, because those models are law rather than suggestion. It owns entity identity and naming, cardinality and join tables, optional relationships and what a null means, ownership and cascade, tenancy columns and scoped uniqueness, additive evolution, and the model library — IAM first: organization, workspace, user, membership, role, permission. Code mechanics live in turystack-backend-pattern; boundaries, consistency and data lifetime live in turystack-architecture-pattern."
---

# turystack-modeling

What exists, and how it relates. Read it before a migration or a new domain.

## What lives here, and what does not

A law belongs here when it is about **shape**: whether something is a table or a
column, which side holds the foreign key, what a null means, which column
carries the tenant, how uniqueness is scoped, and how that shape is allowed to
change.

A law about **boundaries, consistency, failure or lifetime** belongs to
`turystack-architecture-pattern`. A rule about **how the code is written** —
Drizzle syntax, repository methods, decorators — belongs to
`turystack-backend-pattern`.

```text
turystack-architecture-pattern     boundaries, transactions, lifetime, security
└── turystack-modeling             ← this: the shape of the data
    └── turystack-backend-pattern  how that shape is written in this stack
```

**One law, one owner.** Data retention and erasure are `ARC-DAT-*`. Scope coming
from the authenticated context is `ARC-SEC-1`. The permission catalogue is
`ARC-SEC-12`. This skill cites those ids and never restates them; what it adds
is the shape they imply.

## How to use

1. Read `00-overview.md`. It carries the mental model and the five mistakes that
   cost the most, and it is short enough to keep in context.
2. Open the law section your change touches. Read the whole section rather than
   a remembered summary — these rules are dense.
3. If the domain is one Turystack already models, open its model file (`1x-`)
   and follow it exactly. Those models are not examples; the CLI generates them
   literally, and a project that diverges from one has to say why in writing.
4. Then read `turystack-backend-pattern` for how to write it here.

## Routing

| Your change | Read |
| --- | --- |
| A new table, or deciding whether something *is* a table | `01-entities.md` |
| A foreign key, a join table, an optional relationship, a cascade | `02-relationships.md` |
| Anything a customer owns, any uniqueness rule, any cross-customer read | `03-scoping.md` |
| A migration on a model that already has rows, or flipping a cardinality | `04-evolution.md` |
| Sign-in, organizations, workspaces, roles, permissions, OTP | `10-model-iam.md` |

## How a section is written

- **Concept** — what the section governs.
- **Invariants** — the law, one row per rule, each with a stable id
  `<NAMESPACE>-n`. **This table is what a review binds to.**
- Prose explaining why the non-obvious rules exist, because a rule you
  understand survives a refactor and a rule you memorised does not.
- **Never do** — the same law as violations, so the shape is recognisable in a
  diff.

A model file (`1x-`) adds two things: the **tables**, column by column, and the
**invariants that are true of this model specifically** — the ones a generic
modelling rule cannot express, like "a role carrying a `backoffice:` permission
attaches only to the platform organization".

Ids are stable across versions. Every section opens with a **Rules defined
here** line naming the ids it owns.
