# key-too-long

| | |
|---|---|
| **Title** | Invalid Idempotency Key |
| **HTTP status** | `400 Bad Request` |
| **`type`** | `/errors/key-too-long` |

## What triggers it

The `Idempotency-Key` request header is longer than the maximum of **255
characters**. This is checked before the request body is read and before any
store call, so the handler never runs.

## Example response

```http
HTTP/1.1 400 Bad Request
Content-Type: application/problem+json
```

```json
{
  "type": "https://eben-vranken.github.io/idempo-docs/errors/key-too-long/",
  "title": "Invalid Idempotency Key",
  "status": 400,
  "detail": "The Idempotency-Key header exceeds the maximum length of 255 characters",
  "instance": "/charge"
}
```

## What the client should do

Use a shorter `Idempotency-Key` — 255 characters or fewer. A UUID (36
characters) or a random token of a few dozen characters is plenty. This is a
client bug, not a transient condition: retrying the same oversized key will fail
the same way.
