# Scoping

**Rules defined here:** `SCP-1` · `SCP-2` · `SCP-3` · `SCP-4` ·
`SCP-5` · `SCP-6` · `SCP-7` — the law is the *Invariants* table.

## Concept

Every Turystack product is multi-tenant from the first migration. This section
is about the column that carries the tenant, and about the two questions that
decide whether a leak between customers is possible at all: *can this row say
who owns it?* and *can this query be written without saying it?*

## Invariants

| ID | Law (one line) | Class | Gate |
| --- | --- | --- | --- |
| SCP-1 | Every product row belongs to exactly one organization, and the model states the path. A table with no path to an organization is either platform-wide (a catalogue) or a modelling error, and the model says which. | constitutional | `manual` |
| SCP-2 | A row read by scope carries `organization_id` directly, even when the value is reachable through a parent. Scoping through joins is how a filter gets forgotten. | constitutional | `manual` |
| SCP-3 | The scope value comes from the authenticated context (`ARC-SEC-1`). A scope column is never populated from, or filtered by, a value the client sent. | constitutional | `manual` |
| SCP-4 | A workspace-scoped row carries both `organization_id` and `workspace_id`. The organization stays because it is the tenant; the workspace narrows within it. | constitutional | `manual` |
| SCP-5 | Every uniqueness rule includes its scope: `unique(organization_id, slug)`, never `unique(slug)`. A globally unique natural key is a deliberate, written exception. | constitutional | `manual` |
| SCP-6 | A scoped read is **one** method, never two. `organizationId` is an optional field of its input schema, and the controller decides whether it is filled (`ARC-SEC-1`): forced from the authenticated profile on a client surface, absent on an internal one. There is no separate cross-organization method. | constitutional | `manual` |
| SCP-7 | A read by id does not filter by scope. It fetches the row, and the entity's guard confirms the scope in the use case (`ENT-1`) — so a row belonging to another organization is refused by the domain rather than hidden by a `WHERE`. | constitutional | `manual` |

## Why the column is denormalised

`invoice_line` belongs to an `invoice`, which belongs to an `order`, which
belongs to an `organization`. The tenant is reachable — three joins away. So the
scoped read is:

```sql
SELECT il.* FROM invoice_line il
JOIN invoice i ON i.id = il.invoice_id
JOIN "order" o ON o.id = i.order_id
WHERE o.organization_id = $1
```

Every one of those joins is a chance to omit the `WHERE`, and omitting it does
not fail: it returns more rows, quietly, in an endpoint that looks like it
works. The failure surfaces as a customer seeing another customer's data.

With `organization_id` on `invoice_line`, the scoped read is one predicate, the
index is one column, and — the part that matters — the shape makes the rule
mechanically checkable: a query against a scoped table with no `organization_id`
predicate is a finding a tool can raise, and that is impossible to check when
the tenant lives three joins away.

The cost is a denormalised column that must be written correctly on insert. That
is a single place, in the repository, versus every read forever.

## What is not scoped

Three kinds of table legitimately have no `organization_id`, and the model must
say which one applies:

- **Catalogues owned by the code.** `permission` is seeded from the source and
  is the same for every customer.
- **The person.** `user` is not scoped: one person can belong to several
  organizations, and that is exactly what `membership` expresses.
- **Platform-wide records.** Rows the operator owns. They live under the
  platform organization rather than under nothing, so `organization_id` is
  present and points there — see `10-model-iam.md`.

Anything else with no path to an organization is unfinished modelling.

## Cross-tenant reads

The backoffice exists to look across organizations, so the ability is real. What
the model does **not** do is give it a method of its own.

There is one read, and the scope is an optional field of its input. Who fills
that field is the controller's decision, and `ARC-SEC-1` already states the
three cases: a client surface forces it from the authenticated profile, an
operator surface forces the operator organization's, and an internal surface
leaves it out because querying across organizations is the point.

```ts
// the read — one method, data verbs only (REP-1), input built from the
// canonical schemas through @turystack/query-dsl (REP-L1)
findMany(input: { organizationId?: string; status?: OrderStatus })
```

```ts
// client surface — the scope is forced, and never read from the request
async list(
  @AuthenticatedProfile() profile: IamProfile,
  @Request(listOrdersRequest) req: RequestInput<typeof listOrdersRequest>,
) {
  return this.listOrders.execute({ ...req.query, organizationId: profile.organizationId })
}

// internal surface — no forced scope; organizationId is an optional query
async list(@Request(listOrdersInternalRequest) req: RequestInput<typeof listOrdersInternalRequest>) {
  return this.listOrders.execute(req.query)
}
```

A second method named for crossing tenants would look safer and be worse: two
implementations of the same query, drifting apart, and a filter that has to be
remembered in one of them. One method with an optional scope keeps a single
query, and moves the decision to the one layer that knows who is asking.

What the model owes this arrangement is `SCP-2` — the column has to be on
the row, or the optional filter has nothing to filter on.

## Reading one row

A read by id is not scoped, and that is deliberate:

```ts
const order = await this.orderRepository.findById(input.orderId)

order.checkOrganization(input.organizationId) // ENT-1 — the entity refuses it
```

Filtering by scope in the query answers "not found" for a row that exists and
belongs to someone else. Fetching it and letting the entity's guard refuse it
answers with the denial the situation actually is, and puts the rule in the
entity where every other invariant already lives. `find` returns one row,
`findMany` returns rows, `findPaginated` returns a page — each taking the input
its own read needs, and none of them growing a scoped variant.

## Never do

```text
❌  findById(id) with no guard after it   returns any customer's row
✅  findById(id) + entity.checkOrganization(scope)        (SCP-7, ENT-1)

❌  unique(slug)                          customer B cannot use the obvious name
✅  unique(organization_id, slug)         (SCP-5)

❌  where: { organizationId: input.organizationId }    from the client
✅  where: { organizationId: ctx.organizationId }      (ARC-SEC-1, SCP-3)

❌  invoice_line with no organization_id, scoped through three joins
✅  invoice_line.organization_id, written on insert       (SCP-2)

❌  findMany() and findManyAcrossOrganizations()   two queries that will drift
✅  findMany({ organizationId?: … }) — the controller fills it, or does not
                                                         (SCP-6, ARC-SEC-1)
```
