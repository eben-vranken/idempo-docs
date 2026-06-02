# Behavior

This page describes how the middleware behaves at the edges: concurrent
duplicates, store failures, panics, and responses that cannot be cached.

## No key passes through

A request with **no** `Idempotency-Key` header is forwarded straight to the
handler with no buffering, no fingerprinting, and no store call. Idempotency
only applies to requests that opt in by sending the header.

An `Idempotency-Key` longer than **255 characters** is rejected with
[`400 key-too-long`](errors/key-too-long.md) before the handler runs.

## Concurrency: 409, not blocking

A duplicate that arrives **while the first request is still in flight** receives
[`409 Conflict`](errors/conflict.md) rather than blocking until the first
finishes. This matches Stripe's behavior: the in-flight claim holds the key
(`StatusPending`), and any concurrent claim for the same key is told the request
is already being handled.

Exactly-once execution under concurrent duplicates is the
[atomicity guarantee](storage/custom.md#the-atomicity-guarantee) of the store:
when 50 identical requests race, exactly one wins the claim and runs the
handler; the other 49 get `409`.

## Replay of completed requests

Once a request has completed and its response is stored, a later request with
the **same key and the same fingerprint** (method, path, and body) replays the
stored response without running the handler. The replayed response carries the
stored status code, the stored headers, and an added `Idempotency-Replayed:
true` header.

If the key is reused with a **different** fingerprint, the request is rejected
with [`422 body-mismatch`](errors/body-mismatch.md).

## Store failures: fail closed, then fail open

The middleware's failure handling depends on **when** the store fails relative
to the handler:

- **Before the handler runs (fail closed).** If the store is unavailable at
  claim time — `Claim` returns an error — the request is rejected with
  [`500 internal-server-error`](errors/internal-server-error.md) and the handler
  does **not** run. A mutation is never executed when the middleware cannot
  guarantee idempotency.

- **After the handler ran (fail open).** If the store fails when persisting the
  result (the `Complete` or `Abandon` call after the handler returns), the
  client still receives its response and the failure is logged via the
  configured [`Logger`](configuration.md#logger). The side effect already
  happened, so the response is not withheld; the cost is that this particular
  response may not be replayable.

## Panics release the claim

If the handler panics, the middleware recovers, calls `Abandon` to release the
claim (so the key can be retried), and then re-panics so the panic still
propagates to the server's normal recovery. The `Abandon` call uses a fresh
context bounded by [`PersistentTimeout`](configuration.md#persistenttimeout).

## When a response is stored vs. abandoned

After the handler returns, the middleware decides whether to cache the response.
It **abandons** the claim (storing nothing, freeing the key for retry) if any of
these hold:

- the connection was **hijacked**,
- the response **overflowed** [`MaxResponseBytes`](configuration.md#maxresponsebytes), or
- the response status code is **`>= 500`**.

Otherwise it **completes** the claim, storing the status code, headers, and body
for replay. Treating `5xx` as abandon-and-retry means a transient server error
does not get cached and replayed to every retry.

## Streaming, hijack, and oversized responses

The middleware wraps the `ResponseWriter` in a recorder that forwards `Flush`
and `Hijack` to the underlying writer:

- **SSE / streaming** — the recorder implements `http.Flusher`, so streaming
  responses such as Server-Sent Events flush through the middleware normally.
- **Connection upgrades** — the recorder implements `http.Hijacker`, so upgrades
  such as WebSockets work. Once a connection is hijacked there is no recordable
  HTTP response, so the request's claim is abandoned and nothing is cached.
- **Oversized responses** — a response larger than `MaxResponseBytes` is still
  sent to the client in full, but the recording buffer is dropped and the claim
  is abandoned, so the response is not cached.

The recorder intentionally does **not** implement `io.ReaderFrom`: a `ReadFrom`
fast path would bypass `Write` and leave the body uncaptured, making it
unreplayable.
