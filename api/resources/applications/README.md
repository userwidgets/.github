# Applications

This minimal design supports creating applications.

## Create application

### `POST /applications`

Creates a new application.

- Application key must be unique in canonical lowercase punycode form.
- Repeating the same create with the same natural key returns `200 OK` with
  the existing application.

Example request:

```http
POST /applications HTTP/1.1
Host: api.userwidgets.example
Authorization: Bearer <user-master-token>
Content-Type: application/json
Accept: application/json

{
  "key": "app.example.com",
  "name": "App Example"
}
```

Example response (`201 Created`):

```http
HTTP/1.1 201 Created
Content-Type: application/json
Location: /applications/app.example.com

{
  "key": "app.example.com",
  "name": "App Example"
}
```

Example idempotent response for `POST` (`200 OK`):

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "key": "app.example.com",
  "name": "App Example"
}
```
