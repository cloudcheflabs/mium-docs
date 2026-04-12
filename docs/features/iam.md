# Identity and Access Management

Mium includes a built-in IAM system that provides authentication, authorization, and access control for the platform.

## Authentication

Mium uses JWT-based authentication (HMAC-SHA256):

- **Username / Password**: Basic authentication with password hashing. All routes except `/health` and `/admin/auth/login` require authentication.
- **JWT Tokens**: Bearer token authentication for all API requests. Configurable TTL via `mium.jwt.ttl.seconds`.
- **Token Refresh**: Tokens can be refreshed without re-authentication.

## Users and Groups

- Create and manage users with passwords and metadata
- Organize users into groups
- Support for organizations as a top-level grouping
- Policies attached to a group apply to all members

## Policy-Based Access Control

Mium uses AWS-style JSON policy evaluation with deny-by-default semantics:

```json
{
  "statements": [
    {
      "effect": "ALLOW",
      "actions": ["CHAT", "CONNECTION:READ"],
      "resources": ["*"]
    },
    {
      "effect": "DENY",
      "actions": ["IAM:*", "KMS:*"],
      "resources": ["*"]
    }
  ]
}
```

- Policies define allowed or denied actions on specific resources
- Policies can be attached to users or groups
- Deny rules take precedence over allow rules
- Deny-by-default: no matching policies means access is denied

## IAM Actions

The IAM action vocabulary covers all platform operations:

| Action Category | Examples |
|----------------|---------|
| Auth | Login, logout, password change, token refresh |
| IAM Admin | User/group/policy/organization CRUD |
| Connection | Connection management |
| KMS | Key management |
| Chat | Chat session access |
| Monitoring | Node topology, metrics, log tailing |

## Management

Users, groups, and policies are managed through the Admin UI (Settings > Admin > IAM) or the REST API. The Admin UI provides a visual interface for:

- Creating and managing users
- Organizing users into groups
- Attaching policies to users and groups
- Viewing effective permissions
