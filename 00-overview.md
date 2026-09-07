# Overview

**Rules defined here:** none — every id below is owned by another section.

## The mental model

A Turystack product is modelled in three layers, and confusing them is the
origin of most of the damage.

**The shape** is what this skill governs: tables, columns, keys, cardinalities,
tenancy. It is the slowest thing in the system to change, because rows already
exist in it. A wrong boundary in code is a refactor; a wrong column in a table
with four million rows is a migration, a backfill, and a window where both
shapes must be true at once.

**The rules** are what the domain enforces on that shape: a second workspace is
refused, a membership cannot outlive its organization, an OTP is consumed once.
Rules live in use cases, not in the schema — but the schema decides which rules
are *possible* to express and which are merely hoped for.

**The presentation** is what an application shows about the shape: a workspace
selector, a role picker. Presentation is never where a rule is enforced
(`ARC-SEC-2`).

The line between the shape and the rules is where the expensive mistakes are.
Encoding a *policy* in the shape is the most common one: a customer who buys the
single-workspace plan does not get a different schema, because next quarter they
buy the other plan and the difference becomes a data migration instead of a
setting. See `EVO-3`.

## The five that cost the most

1. **A policy modelled as a shape.** "This customer only has one workspace" is a
   rule. Model the general case and constrain it. `EVO-3`
2. **A row that cannot say which customer it belongs to.** Reachable in
   principle, through four joins, is not the same as scoped. `SCP-2`
3. **Uniqueness that forgot the tenant.** `unique(slug)` across all customers
   means the second customer cannot use the obvious name — and worse, that a
   lookup by slug can cross a tenant. `SCP-5`
4. **A polymorphic foreign key.** `target_type` + `target_id` buys flexibility
   by giving up every referential guarantee the database offers. `REL-6`
5. **A null with no stated meaning.** `workspace_id IS NULL` is law when the
   model says it means *the whole organization*, and a bug when nobody wrote
   that down. `TAB-7`

## Namespaces

| Namespace | Owns |
| --- | --- |
| `TAB` | What becomes a table; identity, keys, columns, naming, types |
| `REL` | Cardinality, join tables, optional relationships, ownership, cascade |
| `SCP` | Tenancy: which column carries the customer, scoped uniqueness, cross-tenant reads |
| `EVO` | How a model changes once rows exist |
| `IAM` | The identity and access model |

Model namespaces are allocated as models are written: `IAM` today,
`BIL`, `NTF` and the rest as they land. A model file never redefines a
generic rule; it cites it.

## Cited from elsewhere

These are law, and they are owned by another skill. They appear here because a
modelling decision routinely depends on them.

| Owner › Id | What it says |
| --- | --- |
| `turystack-architecture-pattern` › `ARC-SEC-1` | Scope comes from the authenticated context, never from a client field |
| `turystack-architecture-pattern` › `ARC-SEC-3` | Authorization is operation **plus** resource |
| `turystack-architecture-pattern` › `ARC-SEC-12` | A permission identifier comes from the product's single catalogue |
| `turystack-architecture-pattern` › `ARC-DAT-3` | Erasure has a declared path reaching every place the data landed |
| `turystack-architecture-pattern` › `ARC-DAT-4` | Soft delete is a state, not a synonym for erased |
| `turystack-backend-pattern` › `REP-1` | A repository uses data verbs (`find`, `save`, `update`), never business operation names |
| `turystack-backend-pattern` › `REP-L1` | Filters, sorting and pagination go through `@turystack/query-dsl` |
| `turystack-backend-pattern` › `ENT-1` | The entity owns the invariants; the use case delegates to a `checkIf…` guard |
| `turystack-backend-pattern` › `CTL-2` | One HTTP surface per consumer, each with its own prefix and OpenAPI document |
| `turystack-backend-pattern` › `ENT-5` | Field order: PK → FKs → important → less important → booleans → status → timestamps and audit |
| `turystack-backend-pattern` › `SCH-8` | The same order inside the schema |

## How a canonical model is written

Every `1x-model-*.md` has the same five parts, and the order matters because
each part answers a question raised by the one before it.

1. **What it is for** — the product question the model answers, in a paragraph.
   A model whose purpose cannot be stated in a paragraph is two models.
2. **The diagram** — an ASCII sketch of the entities and their cardinalities.
   It is the thing people actually read, so it comes before the tables.
3. **The tables** — column by column, with type, nullability and what a null
   means — including the ones every table shares (`<table>_id`, `created_at`,
   `updated_at`, `deleted_at`, the audit trio). A table you have to combine
   with a legend to know its real shape is a table that gets read wrong.
4. **Invariants** — `<MODEL>-n`, the rules that are true of this model and
   that no generic rule can express.
5. **What the CLI generates** — the exact tables, seeds and rules the scaffold
   writes, so a diff between a generated repository and this file is a bug in
   one of the two.
