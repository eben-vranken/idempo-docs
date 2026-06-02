# conflict

| | |
|---|---|
| **Title** | Status Conflict |
| **HTTP status** | `409 Conflict` |
| **`type`** | `/errors/conflict` |

## What triggers it

Another request with the **same `Idempotency-Key` is still in flight**. The
first request won the claim and holds the key in a pending state; a concurrent
duplicate that claims the same key is told the request is already being handled
(`StatusPending`) and receives `409` instead of blocking. This matches Stripe's
behavior — see [Concurrency](../behavior.md#concurrency-409-not-blocking).

## Example response

```http
HTTP/1.1 409 Conflict
Content-Type: application/problem+json
```

```json
{
  "type": "https://eben-vranken.github.io/idempo-docs/errors/conflict/",
  "title": "Status Conflict",
  "status": 409,
  "detail": "Another request is already handing this request.",
  "instance": "/charge"
}
```

## What the client should do

**Retry after a short delay.** This is a transient condition: the original
request is still running. Once it completes, retrying the same key with the same
request will replay the stored response (with `Idempotency-Replayed: true`)
rather than running the side effect again. Back off briefly between retries.
