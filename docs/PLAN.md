# Zitadel Notify Relay — Implementation Plan

**Linear:** INF-369 (parent: INF-363)

## Overview

Standalone HTTP service that receives Zitadel HTTP provider notification payloads and delivers messages via configured backends (initially AWS SES for email, AWS SNS for SMS).

## Design

### Provider interface

```go
type EmailProvider interface {
    SendEmail(ctx context.Context, to string, subject string, htmlBody string, textBody string) error
}

type SMSProvider interface {
    SendSMS(ctx context.Context, to string, message string) error
}
```

Initial implementations:
- `ses.Provider` — AWS SES v2 SDK
- `sns.Provider` — AWS SNS for SMS

Future:
- `sendgrid.Provider`, `mailgun.Provider`, `smtp.Provider`
- `twilio.Provider` (direct SMS)

### Payload types

Uses `zitadel-operator/pkg/notification/` for typed structs (or local copies until operator publishes the module):
- `EmailNotification` — contextInfo + templateData + args
- `SMSNotification` — contextInfo + templateData + args

### Template rendering

The relay receives Zitadel's pre-rendered `templateData` (title, subject, text, URL, colors). Options:
1. **Pass-through** — use Zitadel's text directly (simplest)
2. **Custom templates** — apply a local Go template with templateData + args as input (for branding)

Start with pass-through, add custom templates later.

## Implementation steps

1. [ ] Repo skeleton (devbox, Justfile, go.mod, .goreleaser.yaml, GH Actions)
2. [ ] Notification payload types (or import from zitadel-operator/pkg/notification/)
3. [ ] Provider interface definition
4. [ ] AWS SES email provider implementation
5. [ ] AWS SNS SMS provider implementation
6. [ ] HTTP server (POST /email, POST /sms, GET /health)
7. [ ] Config loader (YAML: backend selection + per-backend config)
8. [ ] Helm chart
9. [ ] .goreleaser.yaml (multi-arch binary + ko image + Lambda ZIP)
10. [ ] deploy/example/ (Pulumi Go: Lambda + SES identity + SNS topic + IAM)
11. [ ] Unit tests (mock AWS SDK)
12. [ ] README with usage examples

## Artifacts published on release

- Container image: `ghcr.io/truvity/zitadel-notify-relay` (linux/amd64, linux/arm64)
- Helm chart: `oci://ghcr.io/truvity/charts/zitadel-notify-relay`
- Lambda ZIP: GitHub Release asset (linux/amd64, linux/arm64)
- Raw binaries: GitHub Release asset (linux + darwin, amd64 + arm64)
