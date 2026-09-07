# Relationships

**Rules defined here:** `REL-1` · `REL-2` · `REL-3` · `REL-4` ·
`REL-5` · `REL-6` · `REL-7` — the law is the *Invariants* table.

## Concept

Which side holds the key, when a relationship becomes a table of its own, what
an optional relationship means, and who owns whose lifetime.

## Invariants

| ID | Law (one line) | Class | Gate |
| --- | --- | --- | --- |
| REL-1 | Cardinality is declared in the model, in words, before it is written in a migration. "One organization has many workspaces; a workspace has exactly one organization" is the artefact — the foreign key is its consequence. | constitutional | `manual` |
| REL-2 | One-to-many puts the foreign key on the many side. An array column, a comma-separated string and a JSON list of ids are not relationships. | constitutional | `manual` |
| REL-3 | Many-to-many is an explicit table with its own key (`<table>_id`), its own timestamps, and a unique constraint on the pair. It is named for what it is (`membership`, `role_permission`), never `a_b`. | constitutional | `manual` |
| REL-4 | An optional relationship is a nullable foreign key whose null carries the meaning the model states (`TAB-7`). It is never a second table that exists only to represent absence. | constitutional | `manual` |
| REL-5 | Exactly one parent owns a row's lifetime, and the model names it. Deleting the owner reaches the row; deleting a non-owner never silently takes it. | constitutional | `manual` |
| REL-6 | No polymorphic foreign key. If a row can point at two kinds of parent, it holds two nullable foreign keys and a check constraint that exactly one is set. | constitutional | `manual` |
| REL-7 | A foreign key may cross domain packages only in the direction the packages already depend (`ARC-LAY-1`, `ARC-LAY-4`). Two domains that need to point at each other are one domain, or one of them holds an id without a constraint and reads through the other's API. | constitutional | `manual` |

## Why the join table has its own id

A composite primary key on `(role_id, permission_id)` is smaller and, for a
week, simpler. Then one of these happens, and all of them happen eventually:

- someone needs to know *when* the permission was granted, and to whom;
- an audit trail needs to reference the grant itself;
- the ORM needs a stable handle for the row to update it;
- the pair has to become a triple, and every foreign key pointing at the pair
  has to change shape.

The key costs 16 bytes and buys all four. The unique constraint on the pair is
what preserves the guarantee the composite key was there for — so nothing is
lost, and it is the constraint, not the key, that should have been carrying that
meaning from the start.

## Why polymorphic keys are refused

`comment(target_type, target_id)` looks like it saves four tables. What it
actually does is move referential integrity out of the database and into the
hope that every writer remembers. There is no foreign key, so a deleted parent
leaves orphans; there is no index the planner can use across types; and every
read needs a branch before it can join.

Two nullable foreign keys with `CHECK (num_nonnulls(order_id, invoice_id) = 1)`
keeps the guarantee and reads honestly. When the count of possible parents grows
past three or four, that is the model telling you the child is really its own
concept with its own table per parent, or that the parents share a supertype
that deserves a table.

## Ownership and cascade

Ownership is a modelling statement, not a database setting: *this row cannot
outlive that one*. A `workspace` cannot outlive its `organization`. A
`membership` cannot outlive either its `user` or its `organization` — but it is
**owned** by the organization, because that is the lifetime the product manages.

Cascade follows ownership and nothing else. A foreign key that is not ownership
— `membership.role_id` — restricts instead: a role in use cannot be deleted, and
the product has to say what happens to the memberships first. That refusal is
the point. A cascade there would silently strip people's access.

Erasure of a person is not a cascade from `user`; it is the declared path in
`ARC-DAT-3`, because the data landed in places no foreign key reaches.

## Never do

```text
❌  workspace.member_ids UUID[]          no constraint, no index, no join
✅  membership(user_id, organization_id, workspace_id?)   (REL-2, REL-3)

❌  user_role(user_id, role_id) PK(user_id, role_id)
✅  membership(membership_id, user_id, organization_id, workspace_id?, role_id,
       unique(user_id, organization_id, workspace_id))    (REL-3)

❌  attachment(owner_type, owner_id)     no foreign key, no integrity
✅  attachment(order_id?, invoice_id?) + CHECK exactly one   (REL-6)

❌  ON DELETE CASCADE on membership.role_id     deleting a role strips access
✅  ON DELETE RESTRICT — the product decides what happens first   (REL-5)

❌  user_without_workspace + user_with_workspace   two tables to model a null
✅  workspace_id NULL, meaning stated in the model   (REL-4)
```
