# API conventions

This section defines protocol-level conventions shared by all API operations.

## Protocol and representation

The UserWidgets API uses HTTP and REST-style resource paths. Request and
response bodies use JSON, and JSON field names use `camelCase`.

The initial API does not use a versioned media-type profile. Clients may omit
the `Accept` header and receive the current supported JSON representation. A
client that sends `Accept` uses:

```http
Accept: application/json
```

Media-type profiles and other version-negotiation mechanisms may be introduced
later. Their compatibility rules must be designed before an additional
representation is published.

Requests with a JSON body send a matching `Content-Type` header:

```http
Content-Type: application/json
```

## Resource identifiers

The API uses resource-specific identifier types. Clients treat all identifiers
as opaque in logic and comparisons, except for documented normalization rules.

### Identifier formats

| Resource | Identifier format |
| --- | --- |
| User | Opaque, URL-safe Base64 value without padding, 12 characters |
| Organization | DNS domain key, canonical lowercase punycode |
| Application | DNS domain key, canonical lowercase punycode |
| Role | Opaque, URL-safe Base64 value without padding, 4 characters |
| Permission | Opaque, URL-safe Base64 value without padding, 8 characters |
| API key | Opaque, URL-safe Base64 value without padding, 8 characters |

Role and Permission IDs are scoped to their Application.

Organization and Application keys must be globally unique in canonical form.
They can be renamed, and prior keys are retained as aliases.

Memberships and Application Instances do not have independent identifiers.
They are addressed through their parent User, Organization, and Application
identifiers.

## Collections and pagination

Collection operations return a JSON array directly. The response does not wrap
the array in a `data` property.

Clients request a range with the `Range` header:

```http
Range: offset=25; limit=25
```

The server reports the actual range it returned with `Content-Range`:

```http
Content-Range: offset 25; limit 25; total 240
```

The server may choose a smaller limit than the client requested. It reports the
actual limit in `Content-Range`. The default limit is defined by each resource.

Collection responses include a `Link` header. `rel="first"` identifies the
first range and `rel="next"` identifies the following range when one exists:

```http
Link: <https://.../users?range=0>; rel="first",
      <https://.../users?range=50>; rel="next"
```

The exact target of each link is opaque to the client; clients follow the URL
provided by the server.

An offset beyond the available collection still returns `200 OK` with an empty
array. The `Content-Range` header reports the requested offset and the total
number of items.

## Filtering and ordering

Each resource declares which fields clients may use for filtering and ordering.
Clients use a query parameter matching the documented JSON field name:

```http
GET /users?status=active
```

Repeating the same filter combines those values with OR:

```http
GET /users?status=pending&status=active
```

Different filter fields combine with AND:

```http
GET /users?status=active&name=Alex
```

Clients order a collection by repeating the `sort` parameter. Earlier sort
parameters are higher priority. Prefix a field with `-` for descending order:

```http
GET /users?sort=name&sort=-createdAt
```

Each resource defines a stable default order used when `sort` is omitted.

An unknown field, a non-filterable field used as a filter, or a non-sortable
field used for ordering produces `400 Bad Request`.

## Mutation semantics

### Create

A successful `POST` that creates a resource returns `201 Created`, a `Location`
header identifying the created resource, and the created resource in the
response body.

Each create operation defines its own natural key. Repeating an otherwise
identical create with the same natural key returns `200 OK` and the existing
resource. If the natural key already exists but the request describes a
different resource or incompatible state, the operation returns `409 Conflict`.

### Update and relationship replacement

`PATCH` changes only the fields present in the request. Omitted fields remain
unchanged. A successful `PATCH` returns `200 OK` with the updated resource.

`PUT` on a relationship collection replaces that complete collection. A
successful replacement returns `200 OK` with the resulting relationship
collection.

### Delete

A successful `DELETE` returns `200 OK` with a deletion result describing the
resource that was removed.

Dependency handling for deletion is defined per resource. There is no global
forced-delete mechanism.

## Concurrency

Mutable resource representations expose an opaque `ETag`. Clients can keep that
value and send it in `If-Match` on a later `PATCH`, relationship replacement, or
`DELETE` to avoid overwriting an unseen change.

`If-Match` is optional. When supplied and no longer valid, the server returns
`412 Precondition Failed`.

Clients treat an `ETag` as opaque and must not parse or modify it.
