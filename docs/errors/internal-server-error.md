# internal-server-error

| | |
|---|---|
| **Title** | Internal Server Error |
| **HTTP status** | `500 Internal Server Error` |
| **`type`** | `/errors/internal-server-error` |

## What triggers it

The middleware could not do its job for a reason that is not the client's fault.
There are two paths to this response, both **before** the handler runs (the
middleware fails closed — see
[Store failures](../behavior.md#store-failures-fail-closed-then-fail-open)):

1. **The idempotency store is unavailable at claim time.** `Claim` returns an
   error, so the middleware cannot guarantee idempotency and rejects the request
   rather than running the side effect.
2. **The request body could not be read** for a reason other than exceeding
   [`MaxBodyBytes`](../configuration.md#maxbodybytes) (a body that is too large
   instead returns [`413 content-too-large`](content-too-large.md)).

In both cases the handler does **not** run.

## Example response

```http
HTTP/1.1 500 Internal Server Error
Content-Type: application/problem+json
```

```json
{
  "type": "https://eben-vranken.github.io/idempo-docs/errors/internal-server-error/",
  "title": "Internal Server Error",
  "status": 500,
  "detail": "Our server idempotency store is unavailable.",
  "instance": "/charge"
}
```

The `detail` text depends on which path produced the error:

- Body could not be read: `"Our server idempotency store is unavailable."`
- Claim failed: `"Our server failed parsing the request body."`

!!! warning "Detail wording discrepancy"
    In the current library source these two `detail` strings are effectively
    swapped relative to their cause: the body-read failure reports the store as
    unavailable, while the claim failure reports a body-parsing problem. The
    `type`, `title`, and `500` status are correct and stable; treat the `detail`
    wording as informational only and rely on the status and `type` to classify
    the error.

## What the client should do

**Retry with the same `Idempotency-Key`, with backoff.** This is a server-side
condition and is typically transient (e.g. the store is briefly unreachable).
Because the handler did not run, retrying is safe: the side effect has not
happened. Reusing the same key means that once the request does succeed, further
retries will replay the stored response.
