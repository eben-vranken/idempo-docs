# Postgres store

The Postgres backend is the choice for **durable, ACID** storage: records
survive restarts and exactly-once execution under concurrent duplicates is
enforced by `INSERT ... ON CONFLICT`.

```go
import "github.com/eben-vranken/idempo/pg"
```

## Setup

Run the migration once to create the table and index, then construct the store:

```go
if err := pg.RunMigration(connStr); err != nil { /* ... */ }
store, err := pg.New(connStr, 24*time.Hour, 5*time.Minute)
```

### `RunMigration`

```go
func RunMigration(connStr string) error
```

`RunMigration` opens a temporary pool against `connStr`, creates the table and
index if they do not already exist, and closes the pool. It is safe to call on
every boot.

### Constructor

```go
func New(connStr string, lockTTL time.Duration, retentionTTL time.Duration) (*PostgresStore, error)
```

```go
store, err := pg.New(connStr, 24*time.Hour, 5*time.Minute)
```

- `connStr` — a pgx connection string; the store builds a `pgxpool.Pool` from
  it.
- `lockTTL` — how long an in-flight claim is held.
- `retentionTTL` — how long a completed response stays replayable.

See [the two TTLs](index.md#the-two-ttls) for the full meaning.

`New` returns an `error` (unlike the in-memory and Redis constructors) because
it opens the connection pool eagerly.

## Schema

`RunMigration` creates the following table and index:

```sql
CREATE TABLE IF NOT EXISTS pgStore (
    idempoKey VARCHAR(255) NOT NULL PRIMARY KEY,
    state VARCHAR(20) NOT NULL,
    token VARCHAR(255) NOT NULL,
    bodyHash VARCHAR(255) NULL,
    responseCode INT NULL,
    responseHeaders BYTEA NULL,
    responseBody BYTEA NULL,
    expiryTime TIMESTAMPTZ NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_pgstore_expiry ON pgStore (expiryTime);
```

The `idx_pgstore_expiry` index on `expiryTime` supports the background sweep
that deletes expired rows.

## Atomicity

`Claim` issues an `INSERT ... ON CONFLICT (idempoKey) DO UPDATE ... WHERE
pgStore.expiryTime < now() RETURNING state`. The upsert is atomic: a brand-new
key, or one whose record has expired, is claimed and returns `new`; if the row
already exists and has not expired, the `WHERE` clause suppresses the update and
the statement returns no rows, prompting a follow-up `SELECT` that classifies
the existing record as `pending`, `completed`, or `conflict`. This is the
Postgres implementation of the
[atomicity guarantee](custom.md#the-atomicity-guarantee).

## Token fencing

`Complete` and `Abandon` scope their `UPDATE` / `DELETE` with
`WHERE idempoKey = $1 AND token = $2 AND state = 'pending'`, so a call whose
token does not match the stored claim affects no rows — the
[token-fencing rule](custom.md#token-fencing).

## Expiry

A background goroutine calls `Sweep` every 5 minutes, running
`DELETE FROM pgStore WHERE expiryTime < now()`. Expired records are also
reclaimed lazily by the `Claim` upsert described above. `Sweep` is exported, so
you can also run it on your own schedule.

The repository's `pg/schema.sql` reference file contains the same schema and can
also be applied directly.

## Cleanup

`PostgresStore` has a `Close()` method that stops the sweep goroutine and closes
the connection pool:

```go
store, err := pg.New(connStr, 24*time.Hour, 5*time.Minute)
// ...
defer store.Close()
```
