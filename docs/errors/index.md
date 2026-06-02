# Error reference

When something prevents the middleware from handling an idempotent request, it
responds with an [RFC 9457](https://www.rfc-editor.org/rfc/rfc9457)
`application/problem+json` body:

```json
{
  "type": "https://eben-vranken.github.io/idempo-docs/errors/conflict/",
  "title": "Status Conflict",
  "status": 409,
  "detail": "Another request is already handing this request.",
  "instance": "/charge"
}
```

The fields are:

| Field | Meaning |
|---|---|
| `type` | A URL identifying the error class. It resolves to the matching page in this reference (one page per error below). |
| `title` | A short, human-readable summary of the error class. |
| `status` | The HTTP status code, repeated in the body. |
| `detail` | A human-readable explanation specific to this occurrence. |
| `instance` | The request path that produced the error (`r.URL.Path`). |

The `Content-Type` of these responses is `application/problem+json`.

## The errors

| `type` path | HTTP status | When it happens |
|---|---|---|
| [`/errors/key-too-long`](key-too-long.md) | `400 Bad Request` | `Idempotency-Key` header exceeds 255 characters |
| [`/errors/content-too-large`](content-too-large.md) | `413 Content Too Large` | request body exceeds `MaxBodyBytes` |
| [`/errors/conflict`](conflict.md) | `409 Conflict` | another request with the same key is in flight |
| [`/errors/body-mismatch`](body-mismatch.md) | `422 Unprocessable Entity` | key reused with a different request |
| [`/errors/internal-server-error`](internal-server-error.md) | `500 Internal Server Error` | store unavailable / claim failed |

!!! note "About the `type` URLs"
    In the current library source the `type` URLs are
    `https://demo.com/errors/<code>` placeholders. They are slated to be swapped
    to point at this documentation site, so each `<code>` resolves at
    `/errors/<code>/` here. The HTTP status, title, and detail text on each page
    are quoted verbatim from the library.
