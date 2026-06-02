# idempo

Idempo is a framework-agnostic HTTP middleware for Go that implements the IETF
[Idempotency-Key](https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/)
draft with Stripe-compatible semantics. It makes unsafe requests (payments,
order creation, any mutation) safe to retry: a duplicate request runs its side
effect **at most once** and replays the original response.

It is built on Go's standard `net/http` and works with `chi`, `gin`, `echo`, or
the standard library mux.

## The "at most once" guarantee

A client sends a unique `Idempotency-Key` header with a request. The middleware
records the first response produced for that key and replays it for any later
request that reuses the key, so a retried request runs its side effect **at most
once**.

Exactly-once execution under concurrent duplicates is enforced by the storage
backend (a mutex in memory, an atomic Lua script in Redis, an
`INSERT ... ON CONFLICT` in Postgres) and verified by tests that fire 50
simultaneous identical requests and assert the handler ran once. The whole suite
runs under the race detector in CI.

## How it works

When a request carries an `Idempotency-Key` header, the middleware runs the
following lifecycle:

1. **Claim** the key atomically before the handler runs.
2. If the key is new, it **runs** your handler, then **stores** the response.
3. If the same key arrives again with the same request, it **replays** the
   stored response (adding `Idempotency-Replayed: true`) without running the
   handler again.
4. If the key is still **in flight**, it returns `409 Conflict`.
5. If the key is reused with a **different** request (method, path, or body), it
   returns `422 Unprocessable Entity`.

A request with **no** `Idempotency-Key` header passes straight through to your
handler untouched.

### Request lifecycle


<img alt="lifecycle diagram" src=".\idempo-drawio.png">

- **new** → run the handler, then store the response (`Complete`) or release
  the claim (`Abandon`).
- **completed** → replay the stored response with `Idempotency-Replayed: true`.
- **pending** → `409 Conflict`.
- **conflict** → `422 Unprocessable Entity`.

The two terminal outcomes for a winning claim:

- **Complete** — the handler finished with a cacheable response (status `< 500`,
  not hijacked, not oversized). The response is stored and retained for the
  retention TTL so it can be replayed.
- **Abandon** — the handler panicked, returned `5xx`, hijacked the connection,
  or produced an oversized response. The claim is released so the key can be
  retried.

## Install

```sh
go get github.com/eben-vranken/idempo
```

Continue to [Getting started](getting-started.md) for the quick-start example, or
jump to the [Error reference](errors/index.md) for the `application/problem+json`
responses the middleware emits.

## License

MIT.
