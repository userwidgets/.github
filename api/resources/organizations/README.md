# Organizations

This minimal design supports inviting users to an organization so flows can be
tested end-to-end.

## Create organization

### `POST /organizations`

Creates a new organization.

- Organization key must be unique in canonical lowercase punycode form.
- The authenticated user becomes an `active` member of the created
  organization.
- Repeating the same create with the same natural key returns `200 OK` with
  the existing organization.

Example request:

```http
POST /organizations HTTP/1.1
Host: api.userwidgets.example
Authorization: Bearer <user-master-token>
Content-Type: application/json
Accept: application/json

{
  "key": "acme.example.com",
  "name": "Acme"
}
```

Example response (`201 Created`):

```http
HTTP/1.1 201 Created
Content-Type: application/json
Location: /organizations/acme.example.com

{
  "key": "acme.example.com",
  "name": "Acme"
}
```

Example idempotent response for `POST` (`200 OK`):

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "key": "acme.example.com",
  "name": "Acme"
}
```

## List organization users

### `GET /organizations/{organizationKey}/members`

Lists users (memberships) for an organization.

Example request:

```http
GET /organizations/acme.example.com/members HTTP/1.1
Host: api.userwidgets.example
Authorization: Bearer <user-master-token>
Accept: application/json
```

Example response (`200 OK`):

```http
HTTP/1.1 200 OK
Content-Type: application/json

[
  {
    "user": "alice@example.com",
    "status": "active"
  },
  {
    "user": "new.user@example.com",
    "status": "invited"
  }
]
```

## Invite or add member

### `POST /organizations/{organizationKey}/members`

Creates an organization membership identified by email.

- If no membership exists for the email in the organization, the server creates
  it with status `invited` and returns `201 Created`.
- If a membership already exists for the same organization and email, the
  server returns `200 OK` and the existing membership.

Example request:

```http
POST /organizations/acme.example.com/members HTTP/1.1
Host: api.userwidgets.example
Authorization: Bearer <user-master-token>
Content-Type: application/json
Accept: application/json

{
  "email": "new.user@example.com"
}
```

Example response (`201 Created`):

```http
HTTP/1.1 201 Created
Content-Type: application/json
Location: /organizations/acme.example.com/members/new.user@example.com

{
  "organization": "acme.example.com",
  "user": "new.user@example.com",
  "status": "invited"
}
```

Example idempotent response for `POST` (`200 OK`):

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "organization": "acme.example.com",
  "user": "new.user@example.com",
  "status": "invited"
}
```

## Accept invite by status update

### `PATCH /organizations/{organizationKey}/members/{user}`

Updates membership status.

This minimal flow accepts an invite by setting status to `active`.

Example request:

```http
PATCH /organizations/acme.example.com/members/new.user@example.com HTTP/1.1
Host: api.userwidgets.example
Authorization: Bearer <user-master-token>
Content-Type: application/json
Accept: application/json

{
  "status": "active"
}
```

Example response (`200 OK`):

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "organization": "acme.example.com",
  "user": "new.user@example.com",
  "status": "active"
}
```

