# Storage backends

All backends implement the `idempo.Store` interface, so you can pick one of the
three provided or [write your own](custom.md).

```go
// In-memory: single instance or testing.
import "github.com/eben-vranken/idempo/inmem"
store := inmem.New(24*time.Hour, 5*time.Minute)

// Redis: distributed, high throughput.
import (
	"github.com/eben-vranken/idempo/redis"
	goredis "github.com/redis/go-redis/v9"
)
store := redis.New(&goredis.Options{Addr: "localhost:6379"}, 24*time.Hour, 5*time.Minute)

// Postgres: durable, ACID.
import "github.com/eben-vranken/idempo/pg"
if err := pg.RunMigration(connStr); err != nil { /* ... */ }
store, err := pg.New(connStr, 24*time.Hour, 5*time.Minute)
```

## The two TTLs

Every backend constructor takes the same two `time.Duration` arguments, in this
order:

| Argument | Meaning |
|---|---|
| `lockTTL` | How long an **in-flight** claim is held. While a key is pending, duplicates get `409`. If the owning request never completes (crash, lost connection), the claim expires after `lockTTL` and the key can be claimed again. |
| `retentionTTL` | How long a **completed** response stays replayable. After a request finishes, its stored response is retained for `retentionTTL`; later requests with the same key and request fingerprint replay it. Once it expires, the key is treated as new again. |

In the examples above, an in-flight claim is held for 24 hours and a completed
response is replayable for 5 minutes.

## Choosing a backend

| Backend | Use when | Notes |
|---|---|---|
| [In-memory](in-memory.md) | Single instance or testing | State is lost on restart and not shared across instances. |
| [Redis](redis.md) | Distributed, high throughput | Atomicity via an atomic Lua script. |
| [Postgres](postgres.md) | Durable, ACID | Atomicity via `INSERT ... ON CONFLICT`. |

See [Write your own backend](custom.md) for the `Store` contract if none of
these fit.
