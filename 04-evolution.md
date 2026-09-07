# Evolution

**Rules defined here:** `EVO-1` · `EVO-2` · `EVO-3` · `EVO-4` ·
`EVO-5` — the law is the *Invariants* table.

## Concept

A model with rows in it is a contract with everything already running against
it. This section is how that contract changes without a window where the system
is wrong.

## Invariants

| ID | Law (one line) | Class | Gate |
| --- | --- | --- | --- |
| EVO-1 | A change is additive first: add nullable, write both, backfill, then constrain. A migration that adds a `NOT NULL` column with no default to a populated table is refused. | constitutional | `manual` |
| EVO-2 | A rename is add, copy, switch readers, drop — four deploys, not one `ALTER`. The old name stays until nothing reads it, and "nothing reads it" is verified, not assumed. | constitutional | `manual` |
| EVO-3 | A choice a customer can flip is a policy, not a shape. Model the general case and constrain it with a domain rule; never generate a different schema per plan, per tier or per customer. | constitutional | `manual` |
| EVO-4 | Dropping a column or table requires the erasure path (`ARC-DAT-3`) to be updated in the same change. Data that vanished from the schema did not vanish from the backups, the read models or the exports. | constitutional | `manual` |
| EVO-5 | A catalogue seeded from code (`permission`) is owned by the code. The seed is idempotent, drift between code and table fails a gate, and nobody edits those rows by hand. | constitutional | `manual` |

## Why the policy never becomes a shape

This is the rule that pays for itself the most, so it is worth the example.

A product offers two shapes of account: one organization with a single
workspace, or one organization with many. The tempting move is to model them
differently — no `workspace` table at all in the simple case, or
`organization.workspace_id` instead of `workspace.organization_id`.

What it costs the day a customer upgrades:

- a data migration per customer, run live, on their data;
- two code paths for every read that touches a workspace, forever, because both
  shapes exist in production at once;
- a scope column that means different things in different databases, which is
  the exact condition under which `SCP-2` stops being checkable.

Modelled as a policy instead: `workspace` always exists, always carries
`organization_id`, and the single-workspace account simply has one — created
with the organization. What the setting changes is a domain rule that refuses
the second workspace, and whether the application renders a selector. Upgrading
a customer is a settings change, and the schema never knew there were two
products.

The general form: **if a customer can change it, it is not allowed to be a
shape.** Shapes change with migrations; policies change with a column value.

## Additive, in four steps

```text
1. add          workspace.archived_at TIMESTAMPTZ NULL      deploy, nothing reads it
2. write both   writers set it; readers still use the old signal
3. backfill     one batch, restartable, with a count that proves it finished
4. constrain    readers switch; the old signal is dropped in a later deploy
```

Steps 2 and 3 can share a deploy. Steps 1 and 4 cannot: between them there must
be a moment where both the old and the new shape are true, because a rollback
lands in that moment.

The backfill produces a number, and the number is checked. "It ran" is not
evidence; `0 rows remaining with archived_at IS NULL AND status = 'archived'`
is.

## Seeded catalogues

`permission` exists in the database so foreign keys can point at it, but the
list is in the source: it is the same for every customer, it is deployed with
the code that enforces it, and a row nobody deployed is a permission nobody
implements.

So the seed is idempotent — insert what is missing, update what changed, and
report what is in the table but not in the code. That last one is the important
report: a permission that disappeared from the source while roles still grant it
means people hold access to something that no longer exists, and that is a
finding rather than a cleanup.

## Never do

```text
❌  ALTER TABLE workspace ADD COLUMN kind TEXT NOT NULL     fails on any row
✅  ADD COLUMN kind TEXT NULL → backfill → SET NOT NULL     (EVO-1)

❌  ALTER TABLE user RENAME COLUMN name TO full_name        old readers 500
✅  add full_name → write both → switch → drop name         (EVO-2)

❌  if (plan === 'starter') { /* no workspace table */ }    two schemas
✅  one schema; a rule refuses the second workspace         (EVO-3)

❌  DROP TABLE otp                                           still in exports and backups
✅  drop + update the erasure path in the same change        (EVO-4)

❌  INSERT INTO permission ... run once, by hand
✅  an idempotent seed the deploy runs, and a drift report   (EVO-5)
```
