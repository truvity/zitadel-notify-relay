# Zitadel Notify Relay — Design & Implementation Plan

**Linear:** INF-369 (parent: INF-363)  
**Role:** Standalone HTTP service for Zitadel notification delivery. Follows patterns established by google-group-sync.

---

## Design Decisions

### CLI

- No subcommands, no config file. Binary starts the daemon immediately.
- All configuration via environment variables.
- Bare Go `main` with `--help` and `--version` flags (no urfave/cli, no cobra).
- `signal.NotifyContext` in main — context flows from root through all components.

```bash
zitadel-notify-relay            # reads env vars, starts daemon
zitadel-notify-relay --help     # shows env var documentation
zitadel-notify-relay --version  # prints version
```

### Configuration (env-only)

| Env var | Required | Default | Description |
|---------|----------|---------|-------------|
| `EMAIL_BACKEND` | No | `ses` | Email delivery backend (ses, sendgrid, smtp) |
| `SMS_BACKEND` | No | `sns` | SMS delivery backend (sns, twilio) |
| `AWS_REGION` | Yes (if ses/sns) | — | AWS region for SES/SNS |
| `SES_FROM_ADDRESS` | Yes (if ses) | — | Verified sender email for SES |
| `SES_CONFIGURATION_SET` | No | — | SES configuration set name (optional, for tracking) |
| `SNS_SENDER_ID` | No | — | SNS sender ID for SMS |
| `PORT` | No | `8080` | HTTP server port |
| `HEALTH_PORT` | No | `7070` | Health probe port |
| `LOG_LEVEL` | No | `info` | Log level (debug/info/warn/error) |
| `LOG_FORMAT` | No | `json` | Log format (json/text) |

No config files. No `--config` flag. No YAML. Service reads env vars only.

### API

**Endpoints:**
- `POST /email` — deliver an email notification
- `POST /sms` — deliver an SMS notification
- `GET /health` → `200 OK` (K8s readiness + Lambda Web Adapter)

**Email request (from Zitadel HTTP email provider):**
```json
{
  "contextInfo": {"recipientAddress": "user@example.com", "language": "en"},
  "templateData": {"title": "Verify Email", "text": "Please verify...", "url": "https://..."},
  "args": {"code": "123456"}
}
```

**Email response (success):**
```
HTTP/1.1 200 OK
Content-Type: application/json

{"messageId": "0100018f..."}
```

**SMS request (from Zitadel HTTP SMS provider):**
```json
{
  "contextInfo": {"recipientPhoneNumber": "+1234567890", "language": "en"},
  "templateData": {"text": "Your code is 123456"}
}
```

**Error response (RFC 9457 Problem Details):**
```
HTTP/1.1 502 Bad Gateway
Content-Type: application/problem+json

{
  "type": "https://github.com/truvity/zitadel-notify-relay/problems/delivery-failed",
  "title": "Delivery Failed",
  "status": 502,
  "detail": "SES rejected: Email address is not verified"
}
```

**Design rules:**
- JSON only
- `application/problem+json` for all errors (RFC 9457)
- No Zitadel API calls — this service only receives and delivers

### Authentication

No authentication in the binary. Auth is delegated to the platform:

| Platform | Auth mechanism |
|----------|---------------|
| AWS Lambda | Function URL with `AWS_IAM` auth type |
| Kubernetes | NetworkPolicy — only Zitadel (or its egress proxy) reaches the service |

This keeps the binary simple and deployment-agnostic. Zitadel's HTTP provider configuration includes the target URL — platform auth ensures only Zitadel can call it.

### Provider Interface

```go
type EmailProvider interface {
    SendEmail(ctx context.Context, req EmailRequest) (EmailResponse, error)
}

type SMSProvider interface {
    SendSMS(ctx context.Context, req SMSRequest) (SMSResponse, error)
}
```

**Initial implementations:**
- `ses.Provider` — AWS SES v2 SDK
- `sns.Provider` — AWS SNS for SMS

**Future (add without changing Zitadel config):**
- `sendgrid.Provider`, `mailgun.Provider`, `smtp.Provider`
- `twilio.Provider` (direct SMS)

Provider selection is env-var driven (`EMAIL_BACKEND`, `SMS_BACKEND`). Adding a new backend means implementing the interface and adding it to the provider registry — no changes to HTTP handlers or Zitadel configuration.

### HTTP Framework

- `fiber/v3` — lightweight, fast
- `samber/slog-fiber` — request logging middleware bridging fiber to slog

### Logging

- `slog` (stdlib structured logging)
- `samber/slog-fiber` for HTTP request logging
- Context-aware: `logger.InfoContext(ctx, ...)`
- JSON format by default (CloudWatch / log aggregation)
- `LOG_FORMAT=text` for local development

### Graceful Shutdown

- `signal.NotifyContext` in `main` (SIGTERM, SIGINT)
- Context flows from root to `app.Run(ctx)` to `server.Run(ctx, ...)`
- fiber graceful shutdown with 5s timeout on context cancellation

### Lambda Web Adapter (LWA)

Same pattern as google-group-sync:
1. Lambda runtime starts the binary via `bootstrap`
2. LWA polls `GET /health` on `HEALTH_PORT` until 200
3. Lambda event arrives → LWA converts to HTTP → sends to `localhost:PORT`
4. Binary responds → LWA converts back to Lambda response

Same binary runs unchanged in Lambda and Kubernetes.

---

## Project Structure

```
zitadel-notify-relay/
├── cmd/
│   └── zitadel-notify-relay/main.go # Entry point (bare Go main, --help/--version, signal.NotifyContext)
├── pkg/
│   ├── app/
│   │   └── app.go                   # Wires all components, calls server.Run(ctx)
│   ├── config/
│   │   └── config.go                # Env var loader + validation
│   ├── provider/
│   │   ├── provider.go              # EmailProvider + SMSProvider interfaces
│   │   ├── ses/
│   │   │   ├── ses.go               # AWS SES v2 implementation
│   │   │   └── ses_test.go          # Unit tests (mock SDK)
│   │   └── sns/
│   │       ├── sns.go               # AWS SNS implementation
│   │       └── sns_test.go          # Unit tests (mock SDK)
│   ├── payload/
│   │   ├── email.go                 # Email notification payload types
│   │   └── sms.go                   # SMS notification payload types
│   └── server/
│       ├── server.go                # fiber/v3 app setup + graceful shutdown
│       ├── handler_email.go         # POST /email handler
│       ├── handler_sms.go           # POST /sms handler
│       ├── handler_test.go          # Unit tests (mock providers)
│       └── problem.go               # RFC 9457 problem+json helpers
├── tests/
│   └── integration/
│       ├── main_test.go             # TestMain (start LocalStack, configure providers)
│       ├── email_test.go            # Email delivery via LocalStack SES
│       └── sms_test.go             # SMS delivery via LocalStack SNS
├── charts/
│   └── zitadel-notify-relay/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           ├── deployment.yaml
│           ├── service.yaml
│           └── serviceaccount.yaml
├── deploy/
│   └── example/
│       ├── main.go                  # Pulumi Go: Lambda + SES identity + SNS + IAM + Function URL
│       └── Pulumi.yaml
├── .goreleaser.yaml                 # Multi-arch builds + ko + Lambda ZIP
├── .github/
│   └── workflows/
│       ├── ci.yaml                  # PR: lint + unit test + build (devbox + DeterminateNix)
│       └── release.yaml             # Tag: goreleaser release (devbox + DeterminateNix)
├── Justfile
├── devbox.json
├── .envrc
├── .editorconfig
├── .gitignore
├── .golangci.yml
├── go.mod
├── go.sum
├── LICENSE
├── README.md
└── docs/
    └── PLAN.md                      # This file
```

### Package Responsibilities

| Package | Responsibility |
|---------|---------------|
| `cmd/zitadel-notify-relay` | Entry point: parse `--help`/`--version`, `signal.NotifyContext`, call `app.Run(ctx)` |
| `pkg/app` | Wire all components: load config, create logger, providers, start server |
| `pkg/config` | Load and validate env vars, backend selection |
| `pkg/provider` | `EmailProvider` + `SMSProvider` interfaces, provider registry |
| `pkg/provider/ses` | AWS SES v2 email implementation |
| `pkg/provider/sns` | AWS SNS SMS implementation |
| `pkg/payload` | Zitadel HTTP provider notification payload types |
| `pkg/server` | fiber/v3 HTTP server, routes, handlers, problem+json, graceful shutdown |

---

## Artifacts (published on release)

| Artifact | Location | Architectures |
|----------|----------|---------------|
| Container image | `ghcr.io/truvity/zitadel-notify-relay` | linux/amd64, linux/arm64 |
| Helm chart (OCI) | `oci://ghcr.io/truvity/charts/zitadel-notify-relay` | — |
| Lambda ZIP | GitHub Release asset | linux/amd64, linux/arm64 |
| Raw binaries | GitHub Release asset | linux/amd64, linux/arm64, darwin/amd64, darwin/arm64 |
| Go module | `github.com/truvity/zitadel-notify-relay` | — |

The Lambda ZIP includes the compiled binary renamed to `bootstrap` (the Lambda runtime entry point). LWA layer is added at deployment time — it is not bundled in the ZIP.

---

## Implementation Steps

### Phase 1: Core (minimal working binary)
1. [ ] `pkg/config/config.go` — env var loader + validation (backend selection)
2. [ ] `pkg/payload/email.go` — email notification payload types (from Zitadel HTTP provider)
3. [ ] `pkg/payload/sms.go` — SMS notification payload types
4. [ ] `pkg/provider/provider.go` — `EmailProvider` + `SMSProvider` interfaces
5. [ ] `pkg/provider/ses/ses.go` — AWS SES v2 email delivery
6. [ ] `pkg/provider/sns/sns.go` — AWS SNS SMS delivery
7. [ ] `pkg/server/problem.go` — RFC 9457 error helpers
8. [ ] `pkg/server/handler_email.go` — `POST /email` handler (parse payload → deliver via provider)
9. [ ] `pkg/server/handler_sms.go` — `POST /sms` handler (parse payload → deliver via provider)
10. [ ] `pkg/server/server.go` — fiber/v3 app, slog-fiber middleware, health probe, graceful shutdown
11. [ ] `pkg/app/app.go` — wire config → logger → providers → server
12. [ ] `cmd/zitadel-notify-relay/main.go` — bare Go main, `--help`/`--version`, `signal.NotifyContext`, call `app.Run(ctx)`
13. [ ] Unit tests: handlers (mock providers), config (env parsing), payload parsing

### Phase 2: Testing
14. [ ] `tests/integration/main_test.go` — TestMain (start LocalStack, set up SES identity + SNS topic)
15. [ ] `tests/integration/email_test.go` — email delivery via LocalStack SES (`//go:build integration`)
16. [ ] `tests/integration/sms_test.go` — SMS delivery via LocalStack SNS (`//go:build integration`)

### Phase 3: Release infrastructure
17. [ ] `.goreleaser.yaml` — multi-arch builds (binary: bootstrap, ko image, Lambda ZIP)
18. [ ] `.github/workflows/ci.yaml` — devbox + DeterminateNix, `just check` on PR
19. [ ] `.github/workflows/release.yaml` — devbox + DeterminateNix, goreleaser + helm on tag push
20. [ ] `charts/zitadel-notify-relay/` — Helm chart (Deployment + Service + SA)

### Phase 4: Deployment example
21. [ ] `deploy/example/main.go` — Pulumi Go: Lambda + SES identity + SNS + IAM + Function URL (AWS_IAM)
22. [ ] `deploy/example/Pulumi.yaml`

### Phase 5: Documentation
23. [ ] Update README.md with final env var reference and usage examples
24. [ ] Tag v0.1.0

---

## Testing Strategy

### Unit tests (run in CI)

```bash
go test ./...
```

- Payload parsing (email + SMS notification structs from Zitadel)
- Handler tests (mock providers → verify correct calls)
- Provider selection logic
- Config env var parsing + validation
- Error format tests (problem+json)

### Integration tests (run locally only)

```bash
go test -tags=integration ./tests/integration/...
```

**Requires:**
- Docker (for LocalStack)
- LocalStack Community Edition (free, Apache 2.0) — emulates AWS SES + SNS
- Build tag: `//go:build integration`

**No keyring needed** — LocalStack requires no real AWS credentials.

**Setup:**
```bash
docker run -d --name localstack -p 4566:4566 localstack/localstack
```

**Test scenarios:**
- Send email via SES → LocalStack confirms delivery
- Send SMS via SNS → LocalStack confirms delivery
- Invalid recipient → provider error → 502 + problem+json
- Missing required fields → 400 + problem+json
- Provider unreachable → 502 + problem+json

---

## Deployment Patterns

### Kubernetes (Helm)

```bash
helm install zitadel-notify-relay oci://ghcr.io/truvity/charts/zitadel-notify-relay \
  --set env.AWS_REGION=eu-central-1 \
  --set env.SES_FROM_ADDRESS=noreply@truvity.xyz \
  --set env.EMAIL_BACKEND=ses \
  --set env.SMS_BACKEND=sns
```

Auth: NetworkPolicy restricts access to Zitadel only. No auth logic in the binary.
AWS credentials: IRSA (IAM Roles for Service Accounts) for SES/SNS access.

### AWS Lambda (with Lambda Web Adapter)

- ARM64 (Graviton) for cost efficiency
- Lambda Web Adapter (LWA) layer translates Lambda events → HTTP to localhost:PORT
- Function URL with `AWS_IAM` auth (Zitadel must invoke via IAM-authenticated HTTP)
- IAM execution role with SES:SendEmail + SNS:Publish permissions
- CloudWatch Logs with JSON format
- Health check on `HEALTH_PORT` used by LWA readiness probe

The Pulumi example in `deploy/example/` demonstrates:
- Lambda function with LWA layer (arm64)
- Function URL with AWS_IAM auth type
- `pulumi.NewRemoteArchive(githubReleaseURL)` — downloads ZIP directly from GitHub Release, no S3 intermediary
- SES email identity verification
- SNS topic for SMS
- IAM execution role with SES + SNS permissions
- Env vars: PORT=8080, HEALTH_PORT=7070, AWS_LWA_READINESS_CHECK_PATH=/health, AWS_LWA_READINESS_CHECK_PORT=7070, AWS_LAMBDA_EXEC_WRAPPER=/opt/bootstrap, AWS_LWA_ASYNC_INIT=true

---

## CI/CD

### GitHub Actions

Both CI and Release workflows use:
- **DeterminateSystems/nix-installer-action** — Nix installer for reproducible devbox environment
- **jetify-com/devbox-install-action** (skip-nix-installation: true) — provides Go toolchain, golangci-lint, govulncheck, and goreleaser

### CI workflow (on PR)

```yaml
- uses: DeterminateSystems/nix-installer-action@main
- uses: jetify-com/devbox-install-action@v0.14.0
  with:
    skip-nix-installation: true
- run: devbox run -- just check
```

`just check` runs: build + test + lint + govulncheck

### Release workflow (on tag push)

```yaml
- uses: DeterminateSystems/nix-installer-action@main
- uses: jetify-com/devbox-install-action@v0.14.0
  with:
    skip-nix-installation: true
- run: devbox run -- goreleaser release --clean
# + helm package/push
```

Artifacts published:
- Multi-arch binaries (linux/amd64, linux/arm64, darwin/amd64, darwin/arm64)
- Lambda ZIPs with `bootstrap` binary (linux/amd64, linux/arm64)
- ko container images pushed to `ghcr.io/truvity/zitadel-notify-relay`
- Helm chart OCI pushed to `ghcr.io/truvity/charts/zitadel-notify-relay`

---

## Dependencies

| Package | Purpose |
|---------|---------|
| `github.com/gofiber/fiber/v3` | HTTP server |
| `github.com/samber/slog-fiber` | Request logging middleware (slog ↔ fiber) |
| `github.com/aws/aws-sdk-go-v2/service/sesv2` | AWS SES v2 email delivery |
| `github.com/aws/aws-sdk-go-v2/service/sns` | AWS SNS SMS delivery |

### Not used in this repo

| Package | Reason |
|---------|--------|
| `github.com/urfave/cli/v3` | Bare Go main is sufficient for a daemon with no subcommands |
| `github.com/gofiber/contrib/jwt` | No JWT verification needed — auth is platform-delegated |
| `github.com/zalando/go-keyring` | LocalStack needs no real credentials — no keyring for tests |
| Any config file library | Env-only configuration, no YAML/TOML parsing needed |

---

## GoReleaser Configuration

```yaml
project_name: zitadel-notify-relay

version: 2

builds:
  - id: zitadel-notify-relay
    main: ./cmd/zitadel-notify-relay
    binary: bootstrap
    env:
      - CGO_ENABLED=0
    goos:
      - linux
      - darwin
    goarch:
      - amd64
      - arm64
    ldflags:
      - -s -w -X main.Version={{.Version}}

archives:
  - id: lambda-zip
    formats: [zip]
    name_template: "{{ .ProjectName }}_{{ .Version }}_{{ .Os }}_{{ .Arch }}"

kos:
  - id: zitadel-notify-relay
    build: zitadel-notify-relay
    tags:
      - "{{ .Version }}"
      - "{{ if not .IsSnapshot }}latest{{ end }}"
    platforms:
      - linux/amd64
      - linux/arm64
    bare: true
    preserve_import_paths: false
    base_import_paths: true
    base_image: gcr.io/distroless/static:nonroot
    sbom: none

changelog:
  use: github
  groups:
    - title: Features
      regexp: '^feat'
      order: 0
    - title: Bug Fixes
      regexp: '^fix'
      order: 1
    - title: Other
      order: 999
```

Key points:
- Single build ID, binary named `bootstrap` for Lambda
- Archives: just ZIP with the bootstrap binary
- ko for multi-arch container images (linux/amd64 + linux/arm64)
- Helm chart version from git tag at package time (Chart.yaml has `0.0.0-dev` placeholder)
