# @turystack/modeling

The modelling skill: what becomes a table, how tables relate, how every row is
scoped to a customer, and how a model changes once it has rows in it — plus the
library of canonical models the CLI generates literally.

| File | Owns |
| --- | --- |
| [`SKILL.md`](SKILL.md) | Entry point and routing |
| [`00-overview.md`](00-overview.md) | Mental model, the five costly mistakes, how a model file is written |
| [`01-entities.md`](01-entities.md) | `TAB` — tables, identity, keys, naming, types |
| [`02-relationships.md`](02-relationships.md) | `REL` — cardinality, join tables, optional keys, ownership |
| [`03-scoping.md`](03-scoping.md) | `SCP` — tenancy columns, scoped uniqueness, cross-tenant reads |
| [`04-evolution.md`](04-evolution.md) | `EVO` — additive change, renames, policy vs shape, seeds |
| [`10-model-iam.md`](10-model-iam.md) | `IAM` — identity and access: organization, workspace, user, membership, role, permission |

Models are numbered from `10`. A new canonical model takes the next number and
its own `<NAME>` namespace.

Installed into `.claude/skills` and/or `.codex/skills` by the Turystack CLI.
