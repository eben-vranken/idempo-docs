# In-memory store

The in-memory backend keeps idempotency records in a `map` guarded by a
`sync.Mutex`. It is the simplest option, suited to a **single instance** or
**testing**. State is held in process memory, so it is lost on restart and not
shared across instances.

```go
import "github.com/eben-vranken/idempo/inmem"
```

## Constructor

```go
func New(lockTTL time.Duration, retentionTTL time.Duration) *InMemStore
```

```go
store := inmem.New(24*time.Hour, 5*time.Minute)
```

- `lockTTL` — how long an in-flight claim is held before it can be reclaimed.
- `retentionTTL` — how long a completed response stays replayable.

See [the two TTLs](index.md#the-two-ttls) for the full meaning.

## Atomicity

`Claim` takes the store's mutex for the duration of the call, so when many
requests claim the same key at once, exactly one receives `StatusNew`. This is
the in-memory implementation of the
[atomicity guarantee](custom.md#the-atomicity-guarantee).

## Expiry

A background goroutine sweeps expired keys every 60 seconds. An expired pending
or completed record is also reclaimed lazily: a `Claim` that lands on a key
whose `expiryTime` has passed treats it as new.

## Cleanup

`InMemStore` has a `Close()` method that stops the background sweep goroutine:

```go
store := inmem.New(24*time.Hour, 5*time.Minute)
defer store.Close()
```
