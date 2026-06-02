# body-mismatch

| | |
|---|---|
| **Title** | Unprocessable Entity |
| **HTTP status** | `422 Unprocessable Entity` |
| **`type`** | `/errors/body-mismatch` |

## What triggers it

The `Idempotency-Key` has already been seen, but **this request's fingerprint
differs** from the one originally stored under that key. The fingerprint is a
SHA-256 hash of the request **method**, **path**, and **body**, so reusing a key
with any of those changed produces `StatusConflict` and a `422`.

This protects against accidentally replaying the wrong response: an idempotency
key is meant to identify one specific request, so the same key must always
describe the same request.

## Example response

```http
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/problem+json
```

```json
{
  "type": "https://eben-vranken.github.io/idempo-docs/errors/body-mismatch/",
  "title": "Unprocessable Entity",
  "status": 422,
  "detail": "A request with this key but different body has hit this server already.",
  "instance": "/charge"
}
```

## What the client should do

**Use a fresh `Idempotency-Key` for a genuinely new request.** If the request
really is different (different body, path, or method), it should carry its own
unique key. If the request was meant to be the same retry, make sure the
method, path, and body are byte-for-byte identical to the original — then it
will replay instead of conflicting. Retrying with the same mismatched key will
keep returning `422`.
