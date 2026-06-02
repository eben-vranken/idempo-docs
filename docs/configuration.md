# Configuration

The middleware is constructed with `idempo.New(store, opts)`, where `opts` is an
`idempo.Options`:

```go
type Options struct {
	MaxBodyBytes      int64
	MaxResponseBytes  int64
	PersistentTimeout time.Duration
	Logger            *slog.Logger
}
```

Every field is optional. A zero value (`0` for the numeric and duration fields,
`nil` for the logger) is replaced with the default shown below, so
`idempo.New(store, idempo.Options{})` is valid and uses all defaults.

```go
idempo.New(store, idempo.Options{
	MaxBodyBytes:      1 << 20,           // request body buffered for fingerprinting (default 1 MiB)
	MaxResponseBytes:  1 << 20,           // response buffered for caching; larger responses pass through uncached (default 1 MiB)
	PersistentTimeout: 10 * time.Second,  // timeout for the store call after the handler returns (default 10s)
	Logger:            slog.Default(),    // structured logging; defaults to slog.Default()
})
```

## Fields

### `MaxBodyBytes`

- **Type:** `int64`
- **Default:** `1 << 20` (1 MiB)
- **Effect:** The maximum request body size that is buffered to compute the
  request fingerprint. The body is read through an `http.MaxBytesReader` capped
  at this size; a request whose body exceeds it is rejected with
  [`413 content-too-large`](errors/content-too-large.md) and the handler does
  not run.

### `MaxResponseBytes`

- **Type:** `int64`
- **Default:** `1 << 20` (1 MiB)
- **Effect:** The maximum response size that is buffered for caching. The
  response is still sent to the client in full, but if it grows past this limit
  the buffer is dropped and the response is **not cached** — the claim is
  abandoned so the key can be retried, and no replay is stored. See
  [oversized responses](behavior.md#streaming-hijack-and-oversized-responses).

### `PersistentTimeout`

- **Type:** `time.Duration`
- **Default:** `10 * time.Second`
- **Effect:** The timeout applied to the store call (`Complete` or `Abandon`)
  made **after** the handler returns. This call runs on a fresh
  `context.Background()` with this timeout, so persisting the result is not tied
  to the request's own context (which may already be cancelled once the response
  is sent). The same timeout bounds the `Abandon` call made when the handler
  panics.

### `Logger`

- **Type:** `*slog.Logger`
- **Default:** `slog.Default()`
- **Effect:** The structured logger used for the middleware's own diagnostics —
  for example, a failure to persist a result after the handler ran (fail-open),
  a failed header marshal, or a failure to write a problem response. It does not
  log normal request handling.
