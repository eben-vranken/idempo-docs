# content-too-large

| | |
|---|---|
| **Title** | Content Too Large |
| **HTTP status** | `413 Content Too Large` |
| **`type`** | `/errors/content-too-large` |

## What triggers it

The request body exceeds [`MaxBodyBytes`](../configuration.md#maxbodybytes)
(default 1 MiB). The middleware reads the body through an `http.MaxBytesReader`
capped at that size in order to compute the request fingerprint; when the limit
is hit the read fails with an `*http.MaxBytesError` and the request is rejected.
The handler does not run.

## Example response

```http
HTTP/1.1 413 Content Too Large
Content-Type: application/problem+json
```

```json
{
  "type": "https://eben-vranken.github.io/idempo-docs/errors/content-too-large/",
  "title": "Content Too Large",
  "status": 413,
  "detail": "Body request body size was too large",
  "instance": "/charge"
}
```

## What the client should do

Send a smaller body, or ask the operator to raise
[`MaxBodyBytes`](../configuration.md#maxbodybytes). Retrying the same oversized
body will fail identically, so reducing the payload (or increasing the server's
limit) is required before the request can succeed.
