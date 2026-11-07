# Jaeger v2 collector hardened source build

Jaeger 2.23.0 rebuilt from verified upstream source with its UI assets and placed in a shell-less Chainguard static runtime. Published as `mohankrishna999/jaeger-collector:2.23.0-patched` for amd64 and arm64.

Upstream 2.23.0 ships three Go modules with known vulnerabilities. `build/patch-modules.sh` raises them before the build, and the build fails if a pin does not apply:

| Module | Upstream 2.23.0 | `2.23.0-patched` |
|---|---|---|
| `github.com/andybalholm/brotli` | 1.2.2 | 1.2.8 |
| `golang.org/x/crypto` | 0.58.0 | `v0.58.1-0.20261009181718-2f1f834330b2`, the upstream fix for GO-2026-5932 (no tagged release yet) |
| `golang.org/x/net` | 0.61.0 | 0.62.0 |

The default configuration uses Elasticsearch at `ES_SERVER_URL` (default `http://elasticsearch:9200`). Mount a complete Jaeger v2 configuration over `/cmd/jaeger/config.yaml` for another storage backend.

Three fixes in this release:

- Every listener in `config.yaml` binds `0.0.0.0` explicitly: OTLP gRPC 4317 and HTTP 4318, Jaeger gRPC 14250 and Thrift 6831, 6832 and 14268, Zipkin 9411, and the health check on 13133. Without an endpoint Jaeger 2.23 listens on localhost only, so the Service received no traffic and the kubelet probes on 13133 failed.
- `config.yaml` lists the Elasticsearch URL as a YAML block sequence (`server_urls:` followed by `- ${env:ES_SERVER_URL:-http://elasticsearch:9200}`). Jaeger 2.23 rejects the previous one-line form, `server_urls: [${env:...}]`, at startup (`did not find expected ',' or ']'`), because `{` is a flow indicator inside `[...]`. Write your own configurations the same way.
- `jaeger version` reports `v2.23.0`, the release commit and its date, as the official build does. The build now sets them in `internal/version`, where Jaeger 2.23 reads them; before, the binary reported `dev`.

Verified side by side with the official `jaegertracing/jaeger:2.23.0` image running the same configuration on Elasticsearch 8.19: each collector accepted an OTLP/HTTP trace and stored it; both query services listed the same services and operations, returned both traces identically and found them by search. The collectors expose the same 27 metric names, and neither logs errors. Trivy 0.76.2 and Grype 0.121.1 report no findings.

The image runs as UID 10001 with GID 0 and the included manifests satisfy Kubernetes restricted Pod Security and OpenShift restricted SCC requirements.

