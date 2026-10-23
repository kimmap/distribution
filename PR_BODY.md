On a pull-through cache miss, Distribution fetches and streams the complete blob without forwarding the client's `Range` header. A client can therefore receive `200 OK` and the full blob even when the upstream supports partial responses. This is the behavior reported in [#4667](https://github.com/distribution/distribution/issues/4667).

The registry API documents [resumable pulls](https://distribution.github.io/distribution/spec/api/#resumable-pull) as a use case for Range requests. With the current cache-miss path, clients cannot use the upstream's byte-range support for partial or resumed downloads while the blob is absent from the local cache. Supporting that case would make partial retrieval available before the cache is populated.

This change adds `proxy.rangepassthrough`, also configurable through `REGISTRY_PROXY_RANGEPASSTHROUGH`. It defaults to `false`, preserving the existing behavior unless explicitly enabled.

### Behavior

When enabled, a blob `GET` with a non-empty `Range` header that misses the local cache is forwarded through the existing authenticated upstream client. It forwards `Range` and `If-Range`, follows redirects, and streams the upstream status, body, and relevant representation headers:

- `206 Partial Content`, including multipart responses.
- `200 OK` when the upstream ignores `Range` or `If-Range` selects the full representation.
- `416 Range Not Satisfiable`, including `Content-Range` and the response body.

The upstream interprets the ranges; Distribution does not synthesize partial responses, assemble pieces, or verify the whole blob digest on this path. A partial response requires upstream Range support.

These requests do not populate Distribution's blob cache, including the `200` fallback. Each uncached Range request is relayed independently, so Range-only traffic continues to reach the upstream until the blob is cached by an ordinary GET. Ordinary GETs still fill the cache; cached blob serving, storage redirects, and HEAD requests retain their existing paths. The configuration documentation describes these semantics.

The cache bypass is an explicit tradeoff for deployments where a downstream client already caches the content. Dragonfly is one such deployment: it downloads blobs in pieces and caches them, while Distribution provides the registry endpoint, upstream authentication, and manifest handling. Filling Distribution's complete blob cache would require downloading the whole blob even for a partial request. The opt-in mode lets these deployments use the requested upstream ranges without also requiring a complete cache fill in Distribution.

If copying fails after the response starts, the stream is aborted so an error response cannot append JSON to the blob body. A `416` response is excluded from successful blob transfer metrics and pull events.

### Testing

The regression test reproduces a four-byte Range request receiving the complete 16-byte blob with `200` before the change, and verifies the requested bytes with `206` when enabled.

Tests cover configuration defaults and environment overrides, authenticated requests and redirects, single and multipart ranges, `If-Range` and `200` fallbacks, `416`, cancellation, and interrupted streams. They also verify that the pass-through path does not create a cache writer, while ordinary GETs populate the cache and subsequent Range requests are served locally.

Passed locally with Go 1.26.3 on macOS/arm64 and vendored dependencies:

```sh
go test -mod=vendor -short ./configuration ./internal/client/... \
  ./registry/proxy/... ./registry/handlers ./registry/storage ./registry ./notifications
go test -mod=vendor -race ./internal/client ./registry/proxy ./registry \
  -run '^Test(ServeBlobRange|ProxyRange)' -count=1
```

Lint (golangci-lint 2.9.0, zero issues) and `go vet` passed for the affected packages. Binary builds and the API documentation generation check also passed.

The full short suite (`GOFLAGS=-mod=vendor make test TESTFLAGS=-count=1`) passed in 37 packages; its only failure was `health/checks/TestHTTPChecker`, which expected `200` from `https://www.google.pt` but received `204`. The same failure was reproduced from the unmodified base commit, `cc325ae`. Container-based integration and conformance checks were not run locally.

Separately, a downstream build with the same Range relay logic was exercised with Dragonfly 1.5.7 using 69 unique blobs totaling 11.93 GB. All 1,830 observed target blob GET responses were `206`, and all 91 client/seed cache copies matched the expected SHA-256 digest and size. A subsequent pull reused all 69 blobs from the client's local cache, with no target blob requests observed at Distribution. This provides deployment evidence for the approach; it is separate from testing this upstream commit.

AI assistance was used during implementation, testing, and preparation of this description.
