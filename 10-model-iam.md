# Model · IAM

**Rules defined here:** `IAM-1` … `IAM-14` — the law is the *Invariants*
table. Generic modelling rules are cited, never restated.

## What it is for

Who a person is, which customers they act for, and what they are allowed to do
there. Every Turystack product is born with this model: the `auth` audience
proves identity, and every other audience reads its scope and its permissions
from here. It is the one model the CLI generates literally — a generated
repository that differs from this file is a bug in one of the two.

It answers three questions, and keeping them separate is the whole design.
*Who are you* is `user`, and it is not scoped to anyone. *Where are you acting*
is `membership`, which is the only place a person meets an organization. *What
may you do there* is `role` → `permission`, resolved at sign-in into the session.

## The model

```text
organization ──┬──< workspace
  kind:        ├──< membership ─── user ──┬──< user_social_identity
  CUSTOMER     │         │                └──< otp
  PLATFORM     └──< invitation
                          │
                    role ─┘   both point at the same role
                     │
                     └──< role_permission >── permission
                                              key: <audience>:<res>.<act>

role.kind   ENVIRONMENT   every organization, shipped by the product
            ORGANIZATION  one organization, written by the customer
            BACKOFFICE    the platform only
```

Read the nullable foreign keys as sentences, because they are where the model
does its work:

- `membership.workspace_id IS NULL` — this role applies across the whole
  organization.
- `role.organization_id IS NULL` — the role is not one customer's; `kind` says
  whether it is shipped to every organization or reserved for the platform.
- `invitation.user_id IS NULL` — nobody has accepted it yet, and the person may
  not exist in `user` at all.

## How to read the tables

Every column is listed, including the ones every table shares — a table you have
to combine with a legend to know its real shape is a table that gets read wrong.

The shared ones are: the key `<table>_id` (`TAB-2`), the timestamps
`created_at` and `updated_at` (`TAB-3`), `deleted_at` where withdrawal is a
state rather than an erasure (`ARC-DAT-4`), and the audit trio `created_by`,
`updated_by`, `deleted_by` where a person writes the row (`TAB-9`).
`@turystack/nestjs-database` stamps `created_by` and `updated_by` from the
acting principal wherever the columns exist; the domain writes `deleted_by` when
it withdraws the row. A table that deliberately omits a block says so under it,
with the reason.

Columns are in the order `ENT-5` and `SCH-8` require — **PK → FKs → important →
less important → booleans → status → timestamps and audit** — so this file, the
schema and the entity read the same way top to bottom.

Every column whose values are listed is `text` with the set closed by a Zod enum
in the schema, never a database enum type (`TAB-4`). The JSON under each
table is the row as a repository returns it: camel case, same order, same
columns.

## The tables

### `user`

The person. Not scoped to an organization (`SCP-1`, third exception).

| Column | Type | Null | Meaning |
| --- | --- | --- | --- |
| `user_id` | uuid | no | UUIDv7 (`TAB-2`) |
| `name` | text | no | |
| `email` | text | no | Lower-cased by the domain before every write; unique globally (`IAM-10`) |
| `email_verified_at` | timestamptz | yes | Null means unverified |
| `phone` | text | yes | E.164, `+` and digits. Null means the person gave none |
| `phone_verified_at` | timestamptz | yes | Null means unverified — independent of the e-mail (`IAM-11`) |
| `password_hash` | text | yes | Null means no password: this person signs in socially or by code (`IAM-8`) |
| `password_changed_at` | timestamptz | yes | A session issued before this instant is refused. Null means a password was never set |
| `locale` | text | no | BCP 47. The language an OTP, an invitation and a notification are written in |
| `last_signed_in_at` | timestamptz | yes | Null means never signed in |
| `created_at` | timestamptz | no | |
| `updated_at` | timestamptz | no | |
| `deleted_at` | timestamptz | yes | |
| `created_by` | text | yes | Null when the person signed themselves up |
| `updated_by` | text | yes | |
| `deleted_by` | text | yes | |

`unique(email)` · `unique(phone) where phone is not null`

```json
{
  "userId": "01930f4e-6b21-7c3a-9f10-2c1a5b7d4e00",
  "name": "Ana Ribeiro",
  "email": "ana@acme.com",
  "emailVerifiedAt": "2026-09-01T14:22:10.000Z",
  "phone": "+5511998877665",
  "phoneVerifiedAt": null,
  "passwordHash": "$argon2id$v=19$m=65536,t=3,p=4$c29tZXNhbHQ$...",
  "passwordChangedAt": "2026-09-01T14:20:02.000Z",
  "locale": "pt-BR",
  "lastSignedInAt": "2026-09-07T09:41:33.000Z",
  "createdAt": "2026-09-01T14:20:02.000Z",
  "updatedAt": "2026-09-07T09:41:33.000Z",
  "deletedAt": null,
  "createdBy": null,
  "updatedBy": null,
  "deletedBy": null
}
```

`createdBy` is null because nobody was authenticated: the person signed
themselves up. An operator creating a user from the backoffice leaves their id.

### `user_social_identity`

How a person proves themselves through a provider.

| Column | Type | Null | Meaning |
| --- | --- | --- | --- |
| `user_social_identity_id` | uuid | no | UUIDv7 |
| `user_id` | uuid → `user` | no | Owner (`REL-5`) |
| `provider` | text | no | `APPLE` · `FACEBOOK` · `GOOGLE` · `MICROSOFT` |
| `provider_id` | text | no | The subject the provider asserts. Stable across the person changing their e-mail there |
| `provider_email` | text | yes | As asserted when the link was made. May differ from `user.email`, and is never treated as a verified address |
| `last_used_at` | timestamptz | yes | Last sign-in through this provider. Null means linked but never used |
| `created_at` | timestamptz | no | |
| `updated_at` | timestamptz | no | |
| `created_by` | text | yes | |
| `updated_by` | text | yes | |

`unique(provider, provider_id)` — one provider account maps to one
person.

**No soft delete.** Unlinking a provider removes the row, because a kept row
would still hold `unique(provider, provider_id)` and refuse the re-link.
A deliberate hard delete (`ARC-DAT-4`).

```json
{
  "userSocialIdentityId": "01930f4e-7a02-7b55-8e31-9d4c0f2a6b11",
  "userId": "01930f4e-6b21-7c3a-9f10-2c1a5b7d4e00",
  "provider": "GOOGLE",
  "providerId": "104729183746501928374",
  "providerEmail": "ana@acme.com",
  "lastUsedAt": "2026-09-07T09:41:33.000Z",
  "createdAt": "2026-09-03T11:02:47.000Z",
  "updatedAt": "2026-09-07T09:41:33.000Z",
  "createdBy": "01930f4e-6b21-7c3a-9f10-2c1a5b7d4e00",
  "updatedBy": "01930f4e-6b21-7c3a-9f10-2c1a5b7d4e00"
}
```

### `otp`

A one-time code. Single-use, bounded, and purposeful.

| Column | Type | Null | Meaning |
| --- | --- | --- | --- |
| `otp_id` | uuid | no | UUIDv7 |
| `user_id` | uuid → `user` | no | Owner |
| `purpose` | text | no | `EMAIL_VERIFICATION` · `PASSWORD_RESET` · `SIGN_IN` |
| `channel` | text | no | `EMAIL` · `SMS` — which contact carried it |
| `target` | text | no | The address or number it was sent to, frozen at issue (`IAM-12`) |
| `code_hash` | text | no | Hashed (`TAB-8`) |
| `expires_at` | timestamptz | no | |
| `consumed_at` | timestamptz | yes | Null means still usable (`IAM-6`) |
| `attempts` | integer | no | Failed verifications, default `0`, bounded by the domain |
| `created_at` | timestamptz | no | |
| `updated_at` | timestamptz | no | |

Index on `(user_id, purpose)` where `consumed_at is null` — the lookup every
verification performs.

**No audit** — the actor is the `user_id` already on the row. **No soft delete**
— a spent code is pruned by retention (`ARC-DAT-1`).

```json
{
  "otpId": "01930f50-1c88-7d09-b2a7-5e6f7a8b9c00",
  "userId": "01930f4e-6b21-7c3a-9f10-2c1a5b7d4e00",
  "purpose": "SIGN_IN",
  "channel": "EMAIL",
  "target": "ana@acme.com",
  "codeHash": "$argon2id$v=19$m=19456,t=2,p=1$b3Rwc2FsdA$...",
  "expiresAt": "2026-09-07T09:51:00.000Z",
  "consumedAt": null,
  "attempts": 0,
  "createdAt": "2026-09-07T09:41:00.000Z",
  "updatedAt": "2026-09-07T09:41:00.000Z"
}
```

### `organization`

The tenant. The root of every scope in the product.

| Column | Type | Null | Meaning |
| --- | --- | --- | --- |
| `organization_id` | uuid | no | UUIDv7 |
| `kind` | text | no | `CUSTOMER` · `PLATFORM` (`IAM-2`) |
| `name` | text | no | What a person sees |
| `slug` | text | no | Unique globally — it is the top of the scope tree, so there is nothing to scope it by |
| `workspace_mode` | text | no | `SINGLE` · `MULTI` (`IAM-9`) |
| `status` | text | no | `ACTIVE` · `SUSPENDED`. A suspended organization still authenticates its people and resolves no scope for them |
| `created_at` | timestamptz | no | |
| `updated_at` | timestamptz | no | |
| `deleted_at` | timestamptz | yes | |
| `created_by` | text | yes | Null for the platform organization and for a self-service sign-up |
| `updated_by` | text | yes | |
| `deleted_by` | text | yes | |

`unique(slug)`

```json
{
  "organizationId": "01930f4a-3d10-7f42-a81b-6c2e9d5f4a00",
  "kind": "CUSTOMER",
  "name": "Acme Viagens",
  "slug": "acme-viagens",
  "workspaceMode": "MULTI",
  "status": "ACTIVE",
  "createdAt": "2026-08-12T10:00:00.000Z",
  "updatedAt": "2026-09-01T14:20:02.000Z",
  "deletedAt": null,
  "createdBy": null,
  "updatedBy": "01930f4e-6b21-7c3a-9f10-2c1a5b7d4e00",
  "deletedBy": null
}
```

### `workspace`

A division inside an organization. Present in both workspace modes.

| Column | Type | Null | Meaning |
| --- | --- | --- | --- |
| `workspace_id` | uuid | no | UUIDv7 |
| `organization_id` | uuid → `organization` | no | Owner (`REL-5`) |
| `name` | text | no | |
| `slug` | text | no | Unique within the organization (`SCP-5`) |
| `is_default` | boolean | no | Where a session lands when the person picked no workspace. Exactly one per organization |
| `created_at` | timestamptz | no | |
| `updated_at` | timestamptz | no | |
| `deleted_at` | timestamptz | yes | |
| `created_by` | text | yes | Null for the workspace created with the organization |
| `updated_by` | text | yes | |
| `deleted_by` | text | yes | |

`unique(organization_id, slug)` · `unique(organization_id) where is_default`

```json
{
  "workspaceId": "01930f4b-8e77-7a13-9c40-1f5b2d6e3a00",
  "organizationId": "01930f4a-3d10-7f42-a81b-6c2e9d5f4a00",
  "name": "Operações",
  "slug": "operacoes",
  "isDefault": true,
  "createdAt": "2026-08-12T10:00:00.000Z",
  "updatedAt": "2026-08-12T10:00:00.000Z",
  "deletedAt": null,
  "createdBy": null,
  "updatedBy": null,
  "deletedBy": null
}
```

### `membership`

A person, in an organization, holding a role. The only table that joins a person
to a tenant, and it exists only for someone who has accepted (`IAM-13`).

| Column | Type | Null | Meaning |
| --- | --- | --- | --- |
| `membership_id` | uuid | no | UUIDv7 |
| `user_id` | uuid → `user` | no | |
| `organization_id` | uuid → `organization` | **no** | `IAM-1` |
| `workspace_id` | uuid → `workspace` | **yes** | Null means the whole organization |
| `role_id` | uuid → `role` | no | `ON DELETE RESTRICT` (`REL-5`) |
| `status` | text | no | `ACTIVE` · `SUSPENDED` |
| `created_at` | timestamptz | no | The moment the invitation was accepted |
| `updated_at` | timestamptz | no | |
| `deleted_at` | timestamptz | yes | |
| `created_by` | text | yes | The person who sent the invitation |
| `updated_by` | text | yes | |
| `deleted_by` | text | yes | |

`unique(user_id, organization_id, workspace_id)` — one role per person per scope.

```json
{
  "membershipId": "01930f4c-2b90-7c81-84d2-3a7e1c9f5b00",
  "userId": "01930f4e-6b21-7c3a-9f10-2c1a5b7d4e00",
  "organizationId": "01930f4a-3d10-7f42-a81b-6c2e9d5f4a00",
  "workspaceId": null,
  "roleId": "01930f49-1a55-7e20-b6f3-8d2c4e7a1b00",
  "status": "ACTIVE",
  "createdAt": "2026-09-01T14:25:40.000Z",
  "updatedAt": "2026-09-01T14:25:40.000Z",
  "deletedAt": null,
  "createdBy": "01930f4d-5511-7e02-9b83-4c7a2f1e6d00",
  "updatedBy": null,
  "deletedBy": null
}
```

`workspaceId` is null, so this role applies across the whole organization.

### `invitation`

An offer of access, addressed to an e-mail that may not have a `user` row yet
(`IAM-13`).

| Column | Type | Null | Meaning |
| --- | --- | --- | --- |
| `invitation_id` | uuid | no | UUIDv7 |
| `organization_id` | uuid → `organization` | no | The organization being joined |
| `workspace_id` | uuid → `workspace` | yes | Null means the membership will cover the whole organization |
| `role_id` | uuid → `role` | no | The role the membership will carry. `ON DELETE RESTRICT` |
| `user_id` | uuid → `user` | yes | Null until accepted; then the person who accepted (`IAM-14`) |
| `email` | text | no | Where the offer was sent, lower-cased. Not a foreign key — the person may not exist yet |
| `token_hash` | text | no | Hashed (`TAB-8`). The plaintext exists only in the link |
| `expires_at` | timestamptz | no | |
| `accepted_at` | timestamptz | yes | Null until accepted |
| `revoked_at` | timestamptz | yes | Null unless withdrawn before acceptance |
| `status` | text | no | `PENDING` · `ACCEPTED` · `REVOKED` · `EXPIRED` |
| `created_at` | timestamptz | no | |
| `updated_at` | timestamptz | no | |
| `deleted_at` | timestamptz | yes | |
| `created_by` | text | yes | The person who sent the offer |
| `updated_by` | text | yes | The person who accepted or revoked it |
| `deleted_by` | text | yes | |

`unique(organization_id, email) where status = 'PENDING'` — one open offer per
address per organization.

```json
{
  "invitationId": "01930f4d-c3a1-7f88-9210-6b4e8d1c2f00",
  "organizationId": "01930f4a-3d10-7f42-a81b-6c2e9d5f4a00",
  "workspaceId": null,
  "roleId": "01930f49-1a55-7e20-b6f3-8d2c4e7a1b00",
  "userId": "01930f4e-6b21-7c3a-9f10-2c1a5b7d4e00",
  "email": "ana@acme.com",
  "tokenHash": "$argon2id$v=19$m=19456,t=2,p=1$aW52aXRlc2FsdA$...",
  "expiresAt": "2026-09-08T14:20:02.000Z",
  "acceptedAt": "2026-09-01T14:25:40.000Z",
  "revokedAt": null,
  "status": "ACCEPTED",
  "createdAt": "2026-09-01T14:20:02.000Z",
  "updatedAt": "2026-09-01T14:25:40.000Z",
  "deletedAt": null,
  "createdBy": "01930f4d-5511-7e02-9b83-4c7a2f1e6d00",
  "updatedBy": "01930f4e-6b21-7c3a-9f10-2c1a5b7d4e00",
  "deletedBy": null
}
```

### `role`

A named bundle of permissions. `kind` says who may hold it.

| Column | Type | Null | Meaning |
| --- | --- | --- | --- |
| `role_id` | uuid | no | UUIDv7 |
| `organization_id` | uuid → `organization` | **yes** | Set only when `kind = 'ORGANIZATION'` (`IAM-4`) |
| `kind` | text | no | `ENVIRONMENT` · `ORGANIZATION` · `BACKOFFICE` |
| `key` | text | no | `OWNER`, `ADMIN`, `MEMBER`, `OPERATOR`, or whatever a customer names theirs |
| `name` | text | no | What a person sees |
| `description` | text | yes | Shown in the role editor |
| `created_at` | timestamptz | no | |
| `updated_at` | timestamptz | no | |
| `deleted_at` | timestamptz | yes | |
| `created_by` | text | yes | Null for a seeded role |
| `updated_by` | text | yes | |
| `deleted_by` | text | yes | |

`unique(organization_id, key)`

```json
{
  "roleId": "01930f49-1a55-7e20-b6f3-8d2c4e7a1b00",
  "organizationId": null,
  "kind": "ENVIRONMENT",
  "key": "OWNER",
  "name": "Owner",
  "description": "Full access to the organization, including billing and members.",
  "createdAt": "2026-08-01T00:00:00.000Z",
  "updatedAt": "2026-08-01T00:00:00.000Z",
  "deletedAt": null,
  "createdBy": null,
  "updatedBy": null,
  "deletedBy": null
}
```

`kind` is `ENVIRONMENT`, so `organizationId` is null and every organization can
hand this role out. `createdBy` is null: the row came from the seed.

### `permission`

The catalogue. Seeded from code, never edited by hand (`EVO-5`,
`ARC-SEC-12`).

| Column | Type | Null | Meaning |
| --- | --- | --- | --- |
| `permission_id` | uuid | no | UUIDv7 |
| `key` | text | no | `<audience>:<resource>.<action>` — unique globally (`IAM-5`) |
| `audience` | text | no | `AUTH` · `ADMIN` · `BACKOFFICE` |
| `description` | text | no | Shown in the role editor. Grouping parses the key rather than storing the resource a second time |
| `created_at` | timestamptz | no | |
| `updated_at` | timestamptz | no | |

`unique(key)`

**No audit** and **no soft delete.** Nobody writes these rows; a permission that
disappears from the source is a drift report, not a withdrawal (`EVO-5`).

```json
{
  "permissionId": "01930f48-0c31-7b10-9a22-4b6d8e1f2a00",
  "key": "admin:workspace.create",
  "audience": "ADMIN",
  "description": "Create a workspace inside the organization.",
  "createdAt": "2026-08-01T00:00:00.000Z",
  "updatedAt": "2026-08-01T00:00:00.000Z"
}
```

### `role_permission`

| Column | Type | Null | Meaning |
| --- | --- | --- | --- |
| `role_permission_id` | uuid | no | UUIDv7 |
| `role_id` | uuid → `role` | no | |
| `permission_id` | uuid → `permission` | no | |
| `created_at` | timestamptz | no | |
| `updated_at` | timestamptz | no | |
| `created_by` | text | yes | Who granted it |
| `updated_by` | text | yes | |

`unique(role_id, permission_id)` (`REL-3`)

**No soft delete.** Revoking removes the row, and `created_by` is the record of
who granted it.

```json
{
  "rolePermissionId": "01930f48-9f04-7d66-b013-7c2a5e8d4f00",
  "roleId": "01930f49-1a55-7e20-b6f3-8d2c4e7a1b00",
  "permissionId": "01930f48-0c31-7b10-9a22-4b6d8e1f2a00",
  "createdAt": "2026-08-01T00:00:00.000Z",
  "updatedAt": "2026-08-01T00:00:00.000Z",
  "createdBy": null,
  "updatedBy": null
}
```

## Invariants

| ID | Law (one line) | Class | Gate |
| --- | --- | --- | --- |
| IAM-1 | `membership.organization_id` is NOT NULL. There is no membership outside an organization; the nullable column is `workspace_id`, and its null means the whole organization. | constitutional | `manual` |
| IAM-2 | The platform operator is a member of the organization whose `kind` is `PLATFORM`. Exactly one such organization exists, it is created by the seed, and it is never created through the product. | constitutional | `manual` |
| IAM-3 | A permission in the `backoffice:` namespace attaches only to a role whose `kind` is `BACKOFFICE`, and a `BACKOFFICE` role is granted only inside the platform organization. Enforced at seed time and by a domain rule on role editing. | constitutional | `manual` |
| IAM-4 | `role.kind` decides who may hold the role, and `organization_id` follows from it: `ORGANIZATION` requires one, `ENVIRONMENT` and `BACKOFFICE` forbid one. A customer edits only its own `ORGANIZATION` roles. | constitutional | `manual` |
| IAM-5 | A permission key is `<audience>:<resource>.<action>`. The prefix is what keeps an admin permission from ever satisfying a backoffice check, and it is part of the key rather than a separate column used for filtering. | constitutional | `manual` |
| IAM-6 | An OTP is consumed once: `consumed_at` is set in the same transaction that accepts it, and a consumed or expired code is indistinguishable from a wrong one in the response. Attempts are bounded. | constitutional | `manual` |
| IAM-7 | Sign-in identifies the person, not the tenant. Choosing the organization is a second step that produces the session scope; a person with memberships in three organizations signs in once and picks. | constitutional | `manual` |
| IAM-8 | `password_hash` is null for a person who has no password. It is never a placeholder, a random value, or the hash of an empty string. | constitutional | `manual` |
| IAM-9 | `workspace_mode` is configuration on the organization, never a difference in schema and never a difference in the build (`EVO-3`). Both modes have a `workspace` row; `SINGLE` refuses the second, and the selector is present in every build and hidden at runtime — one bundle serves both kinds of customer. | constitutional | `manual` |
| IAM-10 | `user.email` is unique globally — a deliberate exception to `SCP-5`, because sign-in happens before any organization is known. | constitutional | `manual` |
| IAM-11 | E-mail and phone are verified independently. Verifying one never sets the other's `*_verified_at`, and a code delivered to a channel proves that channel and nothing else. | constitutional | `manual` |
| IAM-12 | `otp.channel` and `otp.target` are frozen at issue. Changing the person's e-mail or phone afterwards does not retarget a code already sent, and a code is accepted only for the target it went to. | constitutional | `manual` |
| IAM-13 | An invitation is its own table, addressed to an e-mail rather than to a user. A `membership` row exists only for someone who accepted, so the member list never shows access that nobody claimed. | constitutional | `manual` |
| IAM-14 | Accepting an invitation writes the `membership` and stamps `accepted_at`, `user_id` and `ACCEPTED` on the invitation, in one transaction. The invitation is never deleted on acceptance: it is the record of who offered the access and when. | constitutional | `manual` |

## Why the operator is a member of an organization

The alternative — `membership.organization_id` nullable, with null meaning
"platform" — was considered and rejected. It buys a row and costs the invariant
that makes scoping checkable: with a nullable tenant, *every* membership query
has to handle a case where there is no tenant, and the compiler cannot tell the
two apart.

Making the platform an organization keeps one shape. Sign-in is one flow. Roles
and permissions are one machine. The operator's own internal teams are
workspaces, with no new code. And `SCP-1` stays literally true: every row
belongs to exactly one organization.

What that costs is a real risk, and `IAM-3` is the guard: if a customer
organization could hold a role with `backoffice:organization.list`, a customer
would read every other customer. `role.kind` is what makes that guard cheap to
check — the question is one column, not a walk through the role's permissions.

## Why the role has a kind

Three kinds of role exist in any product with a backoffice, and encoding them as
"organization_id is null or not" collapses two of them into one:

- **`ENVIRONMENT`** — shipped by the product, available to every organization.
  `OWNER`, `ADMIN`, `MEMBER`. A customer hands them out and cannot edit them.
- **`ORGANIZATION`** — written by one customer, for that customer.
- **`BACKOFFICE`** — the platform's own. `OPERATOR`. This is the only kind that
  may hold a `backoffice:` permission.

Both `ENVIRONMENT` and `BACKOFFICE` have a null `organization_id`, so the null
alone cannot tell them apart — and the difference is exactly the one that
matters, because one is offered to every customer and the other must never be.
`kind` states it, and `IAM-4` binds the null to it.

## Why the invitation is its own table

An invitation is addressed to an **e-mail**, not to a person. The address may
belong to nobody yet, so modelling the offer as a `membership` in a pending
state would require inventing a `user` row for someone who has not agreed to
exist — a half-person that sign-in, the member list and erasure all have to know
about.

Separating them also separates two lifetimes that genuinely differ. An
invitation expires, is revoked, is re-sent. A membership does none of those: it
is active or suspended. Putting both in one row means a status enum that mixes
"has not answered" with "no longer allowed", and every read has to know which
half it is looking at.

What the two share is the destination — organization, workspace, role — and that
is why `invitation` carries the same three foreign keys the membership will get.
Acceptance copies them, and `IAM-14` makes that one transaction.

## Sign-in produces a scope

```text
1. prove identity        password · social · code        → the person
2. list memberships      organizations they belong to    → one, or a choice
3. resolve permissions   role → role_permission → keys   → for that scope
4. issue the session     user_id, organization_id, workspace_id, permissions[]
```

Step 4 is what every other audience reads (`ARC-SEC-1`): the scope on the
session is the only scope the product trusts, and no endpoint accepts an
organization from the client.

A person with one membership never sees step 2 — but the model does not know
that, and the API is the same either way. That is why step 2 is a step and not a
special case.

## The three audiences

| Audience | App | Scope | Permission namespace |
| --- | --- | --- | --- |
| `auth` | `apps/auth` | none — identity only | `auth:` |
| `admin` | `apps/admin` | one organization, from the session | `admin:` |
| `backoffice` | `apps/backoffice` | across organizations | `backoffice:` |

`apps/admin` holds no authentication code: it mounts `<AuthProvider>` and reads
the session. `apps/backoffice` is a separate application rather than a route
inside admin, because a cross-tenant read must not share a bundle, a route tree
or a session shape with a tenant-scoped one.

## What the CLI generates

- `domains/iam` — the ten tables above, their repositories and the use cases for
  sign-up, sign-in (password, social, code), OTP issue and consume, organization
  and workspace creation, invitation send, revoke and accept, and role
  management.
- The seed: the platform organization, the `ENVIRONMENT` roles `OWNER`, `ADMIN`
  and `MEMBER`, the `BACKOFFICE` role `OPERATOR`, and the permission catalogue
  derived from the code (`EVO-5`).
- `apps/auth`, `apps/admin`, `apps/backoffice`, and the three audiences on the
  API.
- The workspace selector in the admin sidebar, always generated and rendered
  only when the signed-in organization's `workspace_mode` is `MULTI` — and the
  domain rule that refuses a second workspace when it is `SINGLE`.
