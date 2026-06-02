# Write your own backend

Any type that satisfies the three-method `idempo.Store` interface can back the
middleware. The in-memory, Redis, and Postgres backends are just three
implementations of this contract.

```go
type Store interface {
	Claim(ctx context.Context, key string, requestHash string, token string) (ClaimResult, error)
	Complete(ctx context.Context, key string, token string, statusCode int, headers []byte, body []byte) error
	Abandon(ctx context.Context, key string, token string) error
}
```

> Store persists idempotency records keyed by the idempotency key and backs the
> middleware with a pluggable storage backend.
>
> Implementations must be safe for concurrent use by multiple goroutines: the
> middleware calls Store from many request handlers at once. Each record follows
> a lifecycle of a single `Claim` followed by exactly one `Complete` or
> `Abandon`, matched by a caller-generated fencing token.

A record's whole life is therefore: **one `Claim`**, then **exactly one
`Complete` or `Abandon`** for the token that won the claim.

## `Claim`

```go
Claim(ctx context.Context, key string, requestHash string, token string) (ClaimResult, error)
```

`Claim` atomically attempts to reserve `key` for the current request.

- `requestHash` is an **opaque fingerprint** of the request; the store only
  compares it and never interprets it.
- `token` is a unique, caller-generated **fencing token** for this attempt; the
  store must persist it so that `Complete` and `Abandon` can later verify
  ownership.

It returns a `ClaimResult` whose `Status` is one of:

| Status | Meaning |
|---|---|
| `StatusNew` | The caller won and now owns the key, held pending under the lock TTL. It must later call `Complete` or `Abandon` with the same token. |
| `StatusPending` | Another request already holds an in-flight claim. |
| `StatusCompleted` | A previous request finished; the response to replay is in `ClaimResult.Code`, `Headers`, and `Body`. |
| `StatusConflict` | The key exists but `requestHash` differs from the stored fingerprint. |

An **expired** pending or completed record may be reclaimed as `StatusNew`.
`Claim` must honor `ctx` cancellation and deadlines.

### The atomicity guarantee

> The atomicity guarantee is the core of the middleware: when many requests
> claim the same key at once, **exactly one of them receives `StatusNew`**.

This is the single most important property to get right. The provided backends
implement it with a mutex (in-memory), an atomic Lua script (Redis), and an
`INSERT ... ON CONFLICT` (Postgres). Your implementation must serialize
competing claims for the same key so that exactly one wins, no matter how many
arrive simultaneously.

### `ClaimResult`

```go
type ClaimResult struct {
	Status  ClaimStatus // the claim outcome, always set
	Code    int         // stored response status code
	Headers []byte      // stored response header set, exactly as written to Complete
	Body    []byte      // stored response body, exactly as written to Complete
}
```

`Code`, `Headers`, and `Body` carry the stored response to replay and are
populated **only** when `Status` is `StatusCompleted`; otherwise they are zero.

## `Complete`

```go
Complete(ctx context.Context, key string, token string, statusCode int, headers []byte, body []byte) error
```

`Complete` records the final response for a finished request and moves the record
to a completed state, retained under the retention TTL.

> `Complete` must be a **no-op** if the stored token does not match `token`, or
> if the record is not pending.

`Complete` must honor `ctx` cancellation and deadlines.

## `Abandon`

```go
Abandon(ctx context.Context, key string, token string) error
```

`Abandon` releases a claim so the key can be retried — for example after the
handler fails, returns a `5xx`, or panics.

> `Abandon` must be a **no-op** if the stored token does not match `token`, so a
> late call cannot release a claim that a newer request now owns.

`Abandon` must honor `ctx` cancellation and deadlines.

## Token fencing

Both `Complete` and `Abandon` are governed by the **fencing-token rule**: each
must compare the token it is given against the token stored by the winning
`Claim`, and **do nothing on a mismatch**.

> This fencing rule prevents a slow request from overwriting a claim that a
> newer request now owns.

Concretely: suppose request A claims a key, stalls, the claim's lock TTL
expires, and request B reclaims the same key with a fresh token. If A finally
wakes up and calls `Complete` (or `Abandon`), the stored token now belongs to B,
so A's call must no-op rather than clobber B's record. This is why `Claim`
generates a token and the store must persist it.

## Compile-time check

The provided backends assert conformance at compile time, and your
implementation can do the same:

```go
var _ idempo.Store = (*MyStore)(nil)
```
