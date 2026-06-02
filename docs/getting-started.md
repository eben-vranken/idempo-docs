# Getting started

## Install

```sh
go get github.com/eben-vranken/idempo
```

## Quick start

The middleware is a plain `func(http.Handler) http.Handler`, so it drops into
any `net/http` server. This example uses the in-memory store:

```go
package main

import (
	"net/http"
	"time"

	"github.com/eben-vranken/idempo"
	"github.com/eben-vranken/idempo/inmem"
)

func main() {
	// lockTTL: how long an in-flight claim is held.
	// retentionTTL: how long a completed response stays replayable.
	store := inmem.New(24*time.Hour, 5*time.Minute)

	mw := idempo.New(store, idempo.Options{})

	mux := http.NewServeMux()
	mux.HandleFunc("POST /charge", func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusCreated)
		w.Write([]byte(`{"status":"charged"}`))
	})

	http.ListenAndServe(":8080", mw.Handler(mux))
}
```

The two arguments to `inmem.New` are the **lock TTL** (how long an in-flight
claim is held before it can be reclaimed) and the **retention TTL** (how long a
completed response stays replayable). See
[Storage backends](storage/index.md) for the other backends.

## Replay demo

Send the same key twice and the second call replays the first response without
running the handler again:

```sh
curl -XPOST localhost:8080/charge -H 'Idempotency-Key: 8f3a...' -d '{"amount":42}'
# 201 Created  {"status":"charged"}

curl -XPOST localhost:8080/charge -H 'Idempotency-Key: 8f3a...' -d '{"amount":42}'
# 201 Created  {"status":"charged"}  +  Idempotency-Replayed: true
```

The replayed response is byte-for-byte the stored one, with an added
`Idempotency-Replayed: true` header so the client can tell a replay from a fresh
execution.

## Use with a router

Because `Handler` is a plain `func(http.Handler) http.Handler`, it drops into
any router, e.g. with `chi`:

```go
r := chi.NewRouter()
r.Use(mw.Handler)
```

The same shape works with `gin`, `echo`, or the standard library mux.

## What's next

- [Configuration](configuration.md) — every `Options` field and its default.
- [Behavior](behavior.md) — concurrency, fail-closed vs fail-open, streaming.
- [Error reference](errors/index.md) — the `problem+json` responses clients can
  receive.
