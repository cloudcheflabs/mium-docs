# Single Sign-On (OIDC, SAML, LDAP)

Mium can hand authentication to an external identity provider, so the cluster
stops being another place that holds passwords and becomes another thing your
directory governs. Three providers are supported — OpenID Connect, SAML 2.0, and
LDAP / Active Directory.

## The two ways in

**The admin console is a browser.** It can be redirected, so it uses the flows
built for that: OIDC Authorization Code with PKCE, or SAML 2.0 Web Browser SSO.
The console shows a button for each provider that is configured, and nothing for
the ones that are not.

**An agent, an SDK or a script is not a browser.** It posts the credential it
already holds and receives the same bearer token a local login returns:

```bash
# Directory credentials
curl -X POST http://mium:8090/auth/sso/ldap/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"alice","password":"<directory password>"}'

# ...or the ordinary login endpoint, which also accepts them
curl -X POST http://mium:8090/auth/login \
  -d '{"username":"alice","password":"<directory password>"}'

# → {"accessToken":"...","username":"alice","federated":true,"idp":"LDAP","groups":["analysts"]}
```

That is the token every `/api/*` route, the SDK and the MCP surface already
accept, so **nothing downstream changes**.

The dedicated directory endpoint exists for a distinction the login form cannot
make. A caller the directory authenticated but whose groups map to nothing is a
configuration problem, not a wrong password — it answers **403** with that
explanation, while a wrong password is **401**. Reporting the first as the second
sends an operator to reset a password that was right.

## What decides permissions

Nothing about the provider does. A federated identity arrives carrying group
names; those are mapped to groups **on this cluster**, and the policies attached
to those groups authorize every request. The evaluation is the same code path as
an access key — only where the group list came from differs.

**No local user account is created.** A federated caller has no password and
nothing to persist; your directory is the record. Creating an account for everyone
who ever signed in would put them all in the IAM list and in every replicated
snapshot, and removing them from the directory would leave the copy behind still
authorizing them.

Instead, a short-lived **federated session** is recorded, and it rides the IAM
snapshot. That matters for a reason specific to Mium: masters and workers both
resolve a caller's policies by name, and neither sees a token. The IAM snapshot is
the one thing that reaches all of them, so a job authorized on a worker resolves
the same identity the console did. A session lasts
`mium.sso.federated.session.seconds` (default 3600); after that the name resolves
to nothing until the caller signs in again.

!!! note "A local account of the same name wins"
    If a stored user and a federated identity share a name, the stored account's
    policies apply. Creating a local user is a deliberate act by an administrator,
    so it is the more specific statement of intent — and resolving the other way
    round would let whoever controls the directory decide what a local account can
    do.

### Group mapping

```properties
mium.sso.group.mappings=db-admins:mium-admins,analysts:mium-readers
```

Left empty, provider group names are used as they are — the common case where the
directory already uses this product's group names. **Once set, the mapping is
exhaustive:** a group not named in it is dropped, so creating a group at the
provider cannot grant access here by itself.

An identity whose groups all map to nothing is refused, not admitted with no
groups. Such a session has no policies and is denied every action, so letting it
in produces someone who is signed in and can do nothing, left to work out why from
permission errors. Set `mium.sso.allow.unmapped.groups=true` to allow it anyway.

## Configuring it

Everything below can be set in the console under **Settings → Single Sign-On**,
which stores it in the `mium_settings` row and applies it on every node without a
restart. The properties file is still read for anything left unset, so a cluster
configured by file keeps working untouched.

Local passwords keep working while SSO is on. Enabling it cannot lock you out.

### OpenID Connect

```properties
mium.sso.oidc.enabled=true
mium.sso.oidc.issuer=https://keycloak.example.com/realms/company
mium.sso.oidc.client.id=mium-console
mium.sso.oidc.client.secret=…
mium.sso.oidc.redirect.uri=https://mium.example.com/auth/sso/oidc/callback
mium.sso.oidc.groups.claim=groups
```

Endpoints are read from the issuer's discovery document, so they are not
configured individually. The ID token's signature is verified against the
provider's published key set, and its issuer, audience and expiry are all checked
— a token issued for a different application is refused even though it is genuine
and correctly signed.

!!! note "The `groups` scope"
    `mium.sso.oidc.scopes` deliberately does **not** include `groups`. It is not a
    standard scope, and a provider that does not define it rejects the whole
    authorization request with `invalid_scope` — so asking for it by default logs
    nobody in. Group membership comes from a claim the provider is configured to
    include.

### SAML 2.0

A SAML integration is an exchange of metadata, not a form-filling exercise.

1. **Download this cluster's metadata** from the console (or
   `GET /api/sso/saml/metadata`) and give it to whoever administers your identity
   provider.
2. **Paste your provider's metadata** into the console. The entity ID, sign-on URL
   and signing certificate are read from it — which beats transcribing three
   fields by hand, where the typos are.

```properties
mium.sso.saml.enabled=true
mium.sso.saml.idp.entity.id=https://idp.example.com/realms/company
mium.sso.saml.idp.sso.url=https://idp.example.com/protocol/saml
mium.sso.saml.idp.certificate=MIIC…
mium.sso.saml.sp.entity.id=mium
mium.sso.saml.sp.acs.url=https://mium.example.com/auth/sso/saml/acs
```

Every assertion is checked four ways, and each one is a real attack if skipped:

| Check | What it prevents |
| --- | --- |
| Signature, against the provider's certificate | An assertion the caller wrote |
| Audience | A genuine assertion for another service logging in here |
| Validity window | A captured assertion replayed forever |
| Issuer | Any provider the caller can reach being trusted |

On top of those, an assertion already used is refused. That record is kept **in
ZooKeeper, not in one master's memory**: an assertion is a bearer document, and a
replay arrives at whichever master the load balancer picks.

**Encrypted assertions and signed requests.** Several providers encrypt assertions
or require the authentication request to be signed. Both need a service-provider
keypair — generate one from the console, then re-import the SP metadata at your
provider so it picks up the new certificate. An encrypted assertion must carry its
own signature: encryption proves who the assertion was *for*, never who wrote it.

**NameID format** is left empty by default, which omits the request entirely and
lets the provider issue whatever it is configured for. Naming one breaks more
integrations than it fixes.

### LDAP / Active Directory

```properties
mium.sso.ldap.enabled=true
mium.sso.ldap.url=ldaps://ad.example.com:636
mium.sso.ldap.bind.dn=cn=svc-mium,ou=service,dc=example,dc=com
mium.sso.ldap.bind.password=…
mium.sso.ldap.user.base.dn=ou=people,dc=example,dc=com
mium.sso.ldap.user.filter=(sAMAccountName={0})
mium.sso.ldap.group.base.dn=ou=groups,dc=example,dc=com
mium.sso.ldap.group.filter=(member={0})
```

Authentication is **search then bind**. A service account finds the user's entry —
their DN is something the product cannot construct, since Active Directory puts
people under `CN=John Doe,OU=Staff,…` where neither component is the login name —
and the password is then checked by binding as that DN.

That second bind *is* the authentication. Reading a password attribute and
comparing it would be wrong even where the directory allows it: only the server
knows how its own hashes are salted, and account lockout, expiry and disabled flags
are enforced on bind and nowhere else.

Group membership is read **both ways**: from the user's `memberOf` and from a
search of the group tree. Directories disagree about which side records it —
OpenLDAP usually keeps it on the group, Active Directory mirrors it onto the user
— and reading only one way silently returns no groups against half the servers in
the field.

!!! warning "Use TLS"
    Without `ldaps://` or `mium.sso.ldap.starttls=true`, the bind password crosses
    the network in the clear.

## Behind a load balancer

Both browser flows work on any master, regardless of which one started them. The
login state — including the PKCE verifier — is sealed with a key every node
derives from `MIUM_MASTER_KEY` and carried in the `state` parameter itself rather
than held in memory on one node. Unsealing it is also what proves this cluster
issued it, which is the login-CSRF check.

Without that, SSO works on a single master and fails on roughly half of all
attempts behind a load balancer — and it fails looking like a problem at the
identity provider.

## Revocation

Disabling someone at the provider stops new logins immediately. Sessions already
issued keep working until they expire — this cluster is not told about the change.
`mium.sso.federated.session.seconds` bounds how long that gap lasts; shorter is
safer.

A federated session cannot be refreshed. `POST /auth/refresh` refuses it and says
so, because renewing would keep someone signed in after the directory disabled
them — the one thing the session lifetime exists to bound.

## See also

- [IAM](iam.md) — the groups and policies a federated identity maps onto.
- [Admin UI](admin-ui.md) — where all of this is configured.
- [Configuration Reference](../configuration/reference.md) — every setting, with its default.
