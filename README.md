# Zitadel Notify Relay

HTTP notification provider relay for [Zitadel](https://zitadel.com). Receives Zitadel HTTP provider notification payloads (email/SMS) and delivers them via configured backends (AWS SES, AWS SNS, etc.).

## What it does

Zitadel's HTTP provider sends notification payloads (verification codes, password resets, user invitations) as JSON POST to a webhook endpoint. This service receives those payloads and delivers the actual messages via real email/SMS providers.

**Why not use Zitadel's built-in SMTP?** When running Zitadel Cloud or self-hosted without direct SMTP access, the HTTP provider + relay pattern gives you:
- Full control over delivery (templates, retry logic, provider choice)
- No SMTP credentials stored in Zitadel
- Audit trail of all notifications
- Provider flexibility (swap SES for SendGrid without touching Zitadel config)

## Architecture

```
[Zitadel] → HTTP Provider (POST /email or /sms) → [zitadel-notify-relay]
                                                          │
                                                          ├─→ AWS SES (email)
                                                          ├─→ AWS SNS (SMS)
                                                          └─→ (future: SendGrid, Mailgun, Twilio, SMTP relay)
```

## Payload format

Zitadel sends a JSON body with three sections:
- `contextInfo` — event type, provider info, recipient
- `templateData` — title, subject, greeting, text, button URL, branding colors
- `args` — user-specific data (code, display name, login names, etc.)

## Configuration

```yaml
# config.yaml
email:
  backend: ses           # ses | sendgrid | smtp (future)
  ses:
    region: eu-central-1
    # credentials from env/IRSA/instance profile

sms:
  backend: sns           # sns | twilio (future)
  sns:
    region: eu-central-1
```

## Deployment

- **Kubernetes**: Helm chart (`oci://ghcr.io/truvity/charts/zitadel-notify-relay`)
- **AWS Lambda**: ZIP archive from GitHub Release (arm64, with Lambda Web Adapter)
- **Pulumi example**: `deploy/example/` shows Lambda + SES/SNS IAM deployment

## Development

```bash
devbox shell          # activates dev environment
just build            # build binary
just test             # run unit tests
just lint             # run linter
just snapshot         # build snapshot release locally
```

## Related

- [truvity/zitadel-operator](https://github.com/truvity/zitadel-operator) — K8s operator that configures EmailProvider/SmsProvider CRDs pointing to this relay
- [truvity/zitadel-rbac-mapper](https://github.com/truvity/zitadel-rbac-mapper) — Groups-to-grants mapping webhook
- [truvity/google-group-sync](https://github.com/truvity/google-group-sync) — Google Workspace group resolver

## License

MIT
