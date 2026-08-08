# Identity and Access Management

Mium includes a built-in IAM system that provides authentication, authorization, and access control for the platform. The model is intentionally close to AWS IAM so operators familiar with that vocabulary can land quickly.

## Authentication

- **Username + password** — basic authentication on `/auth/login`. Passwords are hashed; an admin-forced password change can be required on first login. HTTP `Basic` auth (username:password) is also accepted on any route.
- **JWT bearer tokens** — HMAC-SHA256 signed access tokens (`Authorization: Bearer <jwt>`) used on all API requests. TTL configurable via `mium.jwt.ttl.seconds`. Refresh available via `/auth/refresh`.
- **Access keys** — AWS-style `(accessKeyId, secretAccessKey)` pairs for programmatic / SDK access, with optional expiry. Each key also mints an `MTOK…` shorthand token usable as `Authorization: Token <mtok>`; the raw `AKIA…`/secret can also be presented via HTTP `Basic`.
- **STS** — short-lived credentials issued via `assume-role`-style endpoints for delegated, time-bounded access.

All routes except the public probes `/health`, `/ready`, and the login route `/auth/login` require authentication.

> Endpoint paths below are shown for the default (empty) `mium.admin.context.path`. If you set a context path (e.g. `/admin`), it is prepended to every route.

## Identities and Hierarchy

The IAM model carries:

- **Users** — credentials, profile metadata, group memberships.
- **Groups** — sets of users; policies attached to a group apply to every member.
- **Policies** — JSON documents describing allow/deny rules.
- **Companies / Organizations** — top-level grouping above users for multi-tenant deployments.

## Policy-Based Access Control

Mium uses AWS-style JSON policies with deny-by-default semantics:

```json
{
  "version": "2024-01-01",
  "statements": [
    {
      "effect": "Allow",
      "action": ["SYSTEM:CHAT", "SYSTEM:USE_CONNECTION"],
      "resource": "*"
    },
    {
      "effect": "Deny",
      "action": ["SYSTEM:MANAGE_IAM", "SYSTEM:MANAGE_KMS"],
      "resource": "*"
    }
  ]
}
```

The statement fields are `effect`, `action`, and `resource` — **singular** `action`/`resource` (each may be a single string or an array), matching the `IamPolicy.Statement` model. `effect` is compared case-insensitively (`Allow`/`Deny`/`ALLOW`/`DENY` all work). An optional `sid` and `condition` are also accepted.

- Policies can be attached to users or groups (group policies cascade to members).
- Explicit DENY overrides any ALLOW.
- No matching ALLOW means the request is rejected.

## Action Vocabulary

The shipped action constants cover all platform operations:

| Category | Actions |
|---|---|
| Chat / Agent | `SYSTEM:CHAT`, `SYSTEM:USE_TOOL`, `SYSTEM:MANAGE_AGENT` |
| Connections | `SYSTEM:USE_CONNECTION`, `SYSTEM:READ_CONNECTION`, `SYSTEM:WRITE_CONNECTION` |
| Memory / Prompt | `SYSTEM:READ_MEMORY`, `SYSTEM:WRITE_MEMORY`, `SYSTEM:READ_PROMPT`, `SYSTEM:WRITE_PROMPT` |
| LLM | `SYSTEM:MANAGE_LLM_BACKEND` |
| Admin | `SYSTEM:MANAGE_IAM`, `SYSTEM:MANAGE_KMS`, `SYSTEM:MANAGE_COMPANY`, `SYSTEM:MANAGE_ORG`, `SYSTEM:MANAGE_BACKUP` |

## Management

Users, groups, and policies are managed through the Admin UI (Settings → Administration → IAM) or the REST API under `/api/iam/*`. The UI exposes user / group / policy CRUD, group memberships, policy attachment, and access-key issuance / download. Companies and organizations are the top-level model but are currently **read-only over REST** — only `GET /api/iam/companies` and `GET /api/iam/orgs` are exposed; there are no create/update/delete company/org endpoints.

Representative endpoints (default context path):

- **Auth** — `POST /auth/login`, `POST /auth/refresh`, `POST /auth/change-password`, `GET /auth/whoami`
- **Users/groups/policies** — `/api/iam/users`, `/api/iam/groups`, `/api/iam/policies`, `/api/iam/add-user-to-group`, `/api/iam/remove-user-from-group`, `/api/iam/attach-group-policy`, `/api/iam/detach-group-policy`
- **Self-service** — `PUT /api/iam/users/me` (own profile)
- **Access keys** — `/api/iam/keys`, `GET /api/iam/download-key` (CSV)
- **Companies/orgs (read-only)** — `GET /api/iam/companies`, `GET /api/iam/orgs`
- **STS** — `GET/POST/DELETE /api/sts/sessions`, `POST /api/sts/assume-role`, `POST /api/sts/sessions/clear`, `GET /api/sts/sessions/download`
