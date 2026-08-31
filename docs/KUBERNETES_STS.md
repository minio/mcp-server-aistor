# Kubernetes: MinIO Operator STS and PolicyBinding credentials

Feature request for `mcp-server-aistor`: support short-lived credentials from the MinIO Operator STS API instead of long-lived static access keys.

## Problem

`mcp-server-aistor` authenticates to MinIO using **static access key + secret key** only (`credentials.NewStaticV4` in `internal/minioclient.New`). The published image reads:

- `MINIO_ENDPOINT`
- `MINIO_ACCESS_KEY`
- `MINIO_SECRET_KEY`
- `MINIO_USE_SSL`

There is no support for **short-lived credentials** via the MinIO Operator **STS** endpoint and **PolicyBinding**-issued service account tokens — the same pattern used by `mc` in Kubernetes deployments with `MC_STS_ENDPOINT` / `MC_WEB_IDENTITY_TOKEN_FILE`.

For MCP servers exposed on a cluster gateway, long-lived static keys in Secrets are undesirable. We want each pod to obtain scoped, rotating credentials from the Operator STS API using its projected service account token.

### Evidence (binary `RELEASE.2025-05-28T17-22-30Z`)

- `github.com/miniohq/mcp-server-aistor-go/internal/minioclient.New` → `credentials.NewStaticV4` only
- No references to `NewChainCredentials`, `NewIAM`, `STSWebIdentity`, or `AWS_WEB_IDENTITY_TOKEN_FILE`
- `minio-go` already supports STS/web-identity via the credential chain; the MCP server does not wire it up

## Proposed solution

### 1. Publish the Go source

The GitHub repo currently contains only a README; the module path in release binaries is `github.com/miniohq/mcp-server-aistor-go`. Publishing the source would allow community patches and clearer contribution.

### 2. Add Operator STS / web-identity support in `minioclient.New`

When STS env vars are set, use `credentials.NewChainCredentials` with `credentials.NewIAM` (web identity) **before** falling back to static V4 keys:

```go
func New(ctx context.Context, cfg Config) (*minio.Client, error) {
    creds := credentials.NewChainCredentials([]credentials.Provider{
        &credentials.EnvAWS{},
        &credentials.FileAWSCredentials{},
        &credentials.IAM{
            Client: &http.Client{Transport: secureTransport(cfg.UseSSL)},
        },
    })
    // If chain returns no keys, fall back to MINIO_ACCESS_KEY / MINIO_SECRET_KEY
    ...
}
```

Suggested env vars (aligned with MinIO Operator / `mc` conventions):

| Variable | Purpose |
|----------|---------|
| `MINIO_WEB_IDENTITY_TOKEN_FILE` | Projected SA token (e.g. `/var/run/secrets/sts.min.io/serviceaccount/token`) |
| `MINIO_STS_ENDPOINT` | Operator STS URL (e.g. `https://sts.min.io-operator.svc:4223/sts/minio`) |
| `MINIO_ENDPOINT` | S3 API endpoint (unchanged) |
| `MINIO_USE_SSL` | TLS to S3 API (unchanged) |
| `MINIO_ACCESS_KEY` / `MINIO_SECRET_KEY` | Optional fallback for non-K8s / local dev |

`PolicyBinding` on the Operator side would scope IAM policies (e.g. read/write vs read-only) per ServiceAccount — no long-lived keys in the MCP Deployment.

### 3. Optional: credential refresh

If session tokens expire, document whether the MCP process should restart on token rotation (projected volume refresh) or implement periodic `credentials.Get()` refresh in the minio client wrapper.

## Alternatives considered

| Approach | Drawback |
|----------|----------|
| Static keys in Kubernetes Secrets | Long-lived; broad blast radius if leaked |
| Sidecar that exchanges STS → static env | Extra container; token refresh complexity |
| Binary patch of release image | Fragile; no source to maintain |
| Fork + custom image only | Duplicates maintenance; upstream fix is cleaner |

## Deployment context

Deploying two MCP instances (read/write + read-only) behind Envoy MCP gateways, with Operator `Policy` + `PolicyBinding` already used for `mc` bootstrap jobs in the same cluster. Native StreamableHTTP (`--http`) — no supergateway wrapper.

Happy to test a preview build or contribute a PR if the Go module is published.
