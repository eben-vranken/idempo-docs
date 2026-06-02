# Redis store

The Redis backend is the choice for a **distributed, high-throughput**
deployment: state is shared across every instance that points at the same Redis,
and exactly-once execution under concurrent duplicates is enforced by an atomic
Lua script.

```go
import (
	"github.com/eben-vranken/idempo/redis"
	goredis "github.com/redis/go-redis/v9"
)
```

## Constructor

```go
func New(opt *goredis.Options, lockTTL time.Duration, retentionTTL time.Duration) *RedisStore
```

```go
store := redis.New(&goredis.Options{Addr: "localhost:6379"}, 24*time.Hour, 5*time.Minute)
```

- `opt` — a `*goredis.Options` from `github.com/redis/go-redis/v9`; the store
  creates its own client from it. Use this to set the address, credentials,
  database, TLS, pool sizing, and so on.
- `lockTTL` — how long an in-flight claim is held.
- `retentionTTL` — how long a completed response stays replayable.

See [the two TTLs](index.md#the-two-ttls) for the full meaning.

## Atomicity

Each record is a Redis hash. `Claim` runs an atomic Lua script that, in a single
server-side step, either creates the hash (`state=pending`) and returns `new`,
or inspects the existing hash and returns `pending`, `completed`, or `conflict`.
Because the script is atomic, concurrent claims for the same key resolve to
exactly one `new`. This is the Redis implementation of the
[atomicity guarantee](custom.md#the-atomicity-guarantee).

## Token fencing

`Complete` and `Abandon` are also Lua scripts that first compare the stored
`token` against the caller's token and no-op on mismatch, implementing the
[token-fencing rule](custom.md#token-fencing). `Complete` additionally no-ops if
the record is no longer `pending`.

## Expiry

TTLs are applied with Redis `PEXPIRE`: the `lockTTL` is set when a claim is
created and replaced with the `retentionTTL` when the record completes, so Redis
expires records itself — there is no separate sweep.

## Cleanup

`RedisStore` exposes `Close() error`, which closes the underlying go-redis
client:

```go
store := redis.New(&goredis.Options{Addr: "localhost:6379"}, 24*time.Hour, 5*time.Minute)
defer store.Close()
```
