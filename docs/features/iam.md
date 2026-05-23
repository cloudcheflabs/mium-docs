# Identity and Access Management

Mium includes a built-in IAM system that provides authentication, authorization, and access control for the platform. The model is intentionally close to AWS IAM so operators familiar with that vocabulary can land quickly.

## Authentication

- **Username + password** — basic authentication on `/admin/auth/login`. Passwords are hashed; an admin-forced password change can be required on first login.
- **JWT bearer tokens** — HMAC-SHA256 signed access tokens used on all API requests. TTL configurable via `mium.jwt.ttl.seconds`. Refresh available via `/admin/auth/refresh`.
- **Access keys** — AWS-style `(accessKeyId, secretAccessKey)` pairs for programmatic / SDK access, with optional expiry.
- **STS** — short-lived credentials issued via `assume-role`-style endpoints for delegated, time-bounded access.

All routes except `/health` and `/admin/auth/login` require authentication.

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
  "statements": [
    {
      "effect": "ALLOW",
      "actions": ["SYSTEM:CHAT", "SYSTEM:USE_CONNECTION"],
      "resources": ["*"]
    },
    {
      "effect": "DENY",
      "actions": ["SYSTEM:MANAGE_IAM", "SYSTEM:MANAGE_KMS"],
      "resources": ["*"]
    }
  ]
}
```

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
| Admin | `SYSTEM:MANAGE_IAM`, `SYSTEM:MANAGE_KMS`, `SYSTEM:MANAGE_COMPANY`, `SYSTEM:MANAGE_ORG` |

## Management

Users, groups, policies, companies, and organizations are managed through the Admin UI (Settings → Administration → IAM) or the REST API under `/admin/api/iam/*`. The UI exposes user / group / policy CRUD, group memberships, policy attachment, and access-key issuance / download.
