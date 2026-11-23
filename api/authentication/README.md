# Authentication and authorization

This section defines interactive user authentication and application
authorization token exchange under `/me`.

## Goals

- One cross-application **master token** represents the authenticated User.
- Application access uses shorter-lived **application tokens**.
- A User can sign in once and obtain organization-scoped application tokens for
  seamless organization switching.

## Endpoints

### `GET /me`

`GET /me` is used for login, reauthentication, and master-token renewal.

#### Login mode (native credentials)

Clients authenticate with:

```http
Authorization: Basic <base64-username-password>
```

Success returns:

- a new or renewed master token (in JSON body),
- user fields,
- organizations keyed by organization key, including membership statuses,
- accessible applications keyed by application key.

Example success:

```http
GET /me HTTP/1.1
Host: api.userwidgets.example
Authorization: Basic YWxpY2VAZXhhbXBsZS5jb206c2VjcmV0
Accept: application/json
```

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "token": "<user-master-token>",
  "id": "Ab3DefGh12Jk",
  "name": "Alice Example",
  "emails": [
    "alice@example.com",
    "alice@work.example"
  ],
  "organizations": {
    "acme.example.com": {
      "user": "alice@work.example",
      "status": "active"
    },
    "globex.example.com": {
      "user": "alice@example.com",
      "status": "invited"
    }
  },
  "applications": {
    "support.example.com": {
      "name": "Support Desk"
    },
    "billing.example.com": {
      "name": "Billing"
    }
  }
}
```

Example error (MFA required):

```http
GET /me HTTP/1.1
Host: api.userwidgets.example
Authorization: Basic YWxpY2VAZXhhbXBsZS5jb206c2VjcmV0
Accept: application/json
```

```http
HTTP/1.1 403 Forbidden
Content-Type: application/json

{
  "error": "step_up_required",
  "refresh": "mfa",
  "methods": [
    "totp",
    "webauthn"
  ]
}
```

#### Renewal mode (master token)

Clients call:

```http
Authorization: Bearer <user-master-token>
```

Every successful renewal returns a **renewed master token** in the response
body. The previously issued master token remains valid until its own `exp`.

### `PUT /me`

`PUT /me` creates a new user.

User creation is independent of prior invites. Matching invites are not
automatically linked or activated during creation.

Successful `PUT /me` responses return the same payload shape as successful
`GET /me` responses and include a user master token.

Example:

```http
PUT /me HTTP/1.1
Host: api.userwidgets.example
Content-Type: application/json
Accept: application/json

{
  "name": "Alice Example",
  "emails": [
    "alice@example.com",
    "alice@work.example"
  ],
  "password": "<password>"
}
```

```http
HTTP/1.1 201 Created
Content-Type: application/json

{
  "token": "<user-master-token>",
  "id": "Ab3DefGh12Jk",
  "name": "Alice Example",
  "emails": [
    "alice@example.com",
    "alice@work.example"
  ],
  "organizations": {},
  "applications": {}
}
```

### `POST /me`

`POST /me` is used for MFA submission.

Example request:

```http
POST /me HTTP/1.1
Host: api.userwidgets.example
Authorization: Bearer <user-master-token>
Content-Type: application/json
Accept: application/json

{
  "method": "totp",
  "code": "<mfa-code>"
}
```

Successful `POST /me` responses return the same payload shape as successful
`GET /me` responses:

- `token`,
- user fields (`id`, `name`, `emails`),
- `organizations` keyed by organization key,
- `applications` keyed by application key.

### `GET /me/applications/{applicationKey}`

Returns organizations the User can access for the application, including:

- organization key,
- organization name,
- user (one selected email for that organization membership),
- application token for that organization.

This endpoint is used to prefetch all organization tokens for seamless
in-application organization switching.

Example:

```http
GET /me/applications/app.example.com HTTP/1.1
Host: api.userwidgets.example
Authorization: Bearer <user-master-token>
Accept: application/json
```

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "key": "app.example.com",
  "name": "App Example",
  "organizations": {
    "acme.example.com": {
      "name": "Acme",
      "user": "alice@work.example",
      "token": "<application-organization-token>"
    },
    "globex.example.com": {
      "name": "Globex",
      "user": "alice@example.com",
      "token": "<application-organization-token>"
    }
  }
}
```

## Master token

The master token is private to UserWidgets and is used to mint application
tokens.

### Lifetime and usage windows

- Absolute master token lifetime: **48 hours**.
- Application-token minting window from master token: **first 6 hours**.
- Renewal window: master token can be renewed until the 48-hour deadline.
- Renewal rotation: old and new master tokens are both valid until their own
  expiry.
- Server-side revocation is not supported; logout is client-side token discard.

### Claims

Master token claims:

- `sub`: User ID
- `sid`: session ID
- `amr`: authentication methods reference
- `aut`: authenticated time (custom short claim)
- `iat`: issued-at
- `exp`: expiry
- `iss`: issuer
- `aud`: audience

User profile fields such as name and emails are returned alongside the token,
not inside the token.

## Application token

Application tokens are short-lived and scoped by application + organization.

### Domain identifier rules

- Application and organization identifiers use DNS domains as keys.
- Canonical format is lowercase punycode.
- Identifiers can be renamed; previous values are retained as aliases.
- Responses return keys only.

### Scoping

- `aud` encodes both application and organization as:
  `{applicationKey}/{organizationKey}`.
- Roles and organization display data are returned in endpoint payloads, not in
  token claims.

### Claims

- `sub`: User ID
- `perms`: effective permissions in the current app+org context
- `amr`: authentication methods reference
- `aut`: authenticated time (custom short claim)
- `iat`: issued-at
- `exp`: expiry
- `iss`: issuer
- `aud`: `{applicationKey}/{organizationKey}`

### TTL policy

Application token TTL is:

1. defined per application,
2. optionally overridden per application instance,
3. bounded by platform min/max guardrails.

## Step-up authentication for sensitive operations

Sensitive endpoints can require fresh authentication evidence.

If the token does not satisfy endpoint requirements, the API returns
`403 Forbidden` with:

- `refresh`: one of `login`, `mfa`, or `all`
- `methods`: accepted methods for satisfying the required refresh

Reauthentication entry point is `/me`.

## 2FA transport

2FA inputs are sent in the `POST /me` JSON body, not in `Authorization` and not
in query parameters.

For phishing-resistant MFA, support WebAuthn in the same flow and keep
TOTP/backup codes as fallback methods.

## Deferred scope

- External identity-provider token exchange is deferred and will be documented
  separately.
