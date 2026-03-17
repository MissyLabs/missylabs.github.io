# Secrets Detection

`SecretsDetector` scans text for 37+ credential patterns and prevents accidental secret leakage into logs, audit trails, provider API payloads, and chat output. `SecretCensor` (the redaction layer) replaces detected secrets with `[REDACTED]`.

!!! danger "Secrets in AI contexts"
    LLM agents process user input, tool output, and file contents. Any of these can contain credentials. Without secrets detection, an API key in a config file could be sent to a provider API, logged in plaintext, or echoed back to a chat channel.

## Detection Pipeline

```mermaid
graph LR
    A[Input Text] --> B[SecretsDetector.scan]
    B --> C{Findings?}
    C -->|Yes| D[Sort by position]
    D --> E[Merge overlapping spans]
    E --> F["Replace with [REDACTED]"]
    C -->|No| G[Pass through unchanged]
```

## Detected Credential Types

The detector covers 37+ distinct credential patterns across major cloud providers, SaaS platforms, and common secret formats.

### Cloud Providers

| Type | Pattern | Example Match |
|---|---|---|
| AWS Access Key | `AKIA[0-9A-Z]{16}` | `AKIAIOSFODNN7EXAMPLE` |
| AWS Secret Key | `aws_secret_access_key` + 40 chars | `aws_secret_access_key = wJalrXUtn...` |
| GCP API Key | `AIza[A-Za-z0-9_-]{35}` | `AIzaSyA1234567890abcdefghijklmnop` |
| Azure Key | `AccountKey` + base64 | `AccountKey=abc123...` |

### AI Providers

| Type | Pattern | Example Match |
|---|---|---|
| Anthropic Key | `sk-ant-[A-Za-z0-9_-]{20,}` | `sk-ant-api03-abc123...` |
| OpenAI Key | `sk-(proj-)?[A-Za-z0-9_-]{20,}` | `sk-proj-abc123...` |
| HuggingFace Token | `hf_[A-Za-z0-9]{34,}` | `hf_abcdefghij1234567890...` |

### Code Hosting and CI/CD

| Type | Pattern | Example Match |
|---|---|---|
| GitHub PAT | `ghp_[A-Za-z0-9]{36}` | `ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx` |
| GitHub OAuth | `gho_[A-Za-z0-9]{36}` | `gho_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx` |
| GitLab Token | `glpat-[A-Za-z0-9_-]{20,}` | `glpat-xxxxxxxxxxxxxxxxxxxx` |
| NPM Token | `npm_[A-Za-z0-9]{36}` | `npm_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx` |
| PyPI Token | `pypi-[A-Za-z0-9_-]{50,}` | `pypi-AgEIcHlwaS5v...` |

### Communication and SaaS

| Type | Pattern | Example Match |
|---|---|---|
| Slack Token | `xox[baprs]-...` | `xoxb-123-456-abc` |
| Discord Token | Bot token format | `MTA1...abc.def.ghi...` |
| SendGrid Key | `SG.[base64].[base64]` | `SG.xxxx.yyyy` |
| Stripe Key | `[sr]k_(live\|test)_...` | `sk_live_abc123...` |
| Twilio Key | `SK[a-f0-9]{32}` | `SK1234567890abcdef...` |
| Mailgun Key | `key-[a-f0-9]{32}` | `key-1234567890abcdef...` |

### Infrastructure and Databases

| Type | Pattern | Example Match |
|---|---|---|
| DB Connection String | `postgres://user:pass@host` | `postgres://admin:secret@db:5432` |
| HashiCorp Vault Token | `hvs\|hvb\|hvr.[base64]` | `hvs.abc123...` |
| Databricks Token | `dapi[a-f0-9]{32}` | `dapi1234567890abcdef...` |
| DigitalOcean Token | `dop_v1_[a-f0-9]{64}` | `dop_v1_abcdef...` |
| PlanetScale Token | `pscale_tkn_[base64]` | `pscale_tkn_abc123...` |
| Render Key | `rnd_[A-Za-z0-9]{32,}` | `rnd_abc123...` |
| Fly.io Token | `FlyV1 [base64]` | `FlyV1 abc123...` |

### Cryptographic Material

| Type | Pattern | Example Match |
|---|---|---|
| Private Key | `-----BEGIN (RSA\|EC\|DSA\|OPENSSH) PRIVATE KEY-----` | PEM headers |
| JWT | `eyJ...eyJ...` (3 base64 segments) | `eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOi...` |
| SSH Key Content | `AAAA[BCD][base64]{100+}` | SSH public key body |
| Age Secret Key | `AGE-SECRET-KEY-[A-Z0-9]{59}` | `AGE-SECRET-KEY-1ABC...` |

### Observability and Monitoring

| Type | Pattern | Example Match |
|---|---|---|
| Grafana Token | `glc_[base64]` | `glc_abc123...` |
| Datadog Key | `dd_api_key` + hex | `dd_api_key=abcdef...` |
| New Relic Key | `NRAK-[A-Z0-9]{27}` | `NRAK-ABC123...` |
| PagerDuty Key | `pagerduty_token` + value | `pagerduty_token=abc...` |
| Sentry DSN | `https://[hex]@[host].ingest.sentry.io/[id]` | Full Sentry DSN URL |

### Generic Patterns

| Type | Pattern | Description |
|---|---|---|
| `api_key` | `api_key/apikey` + 20 chars | Generic API key assignments |
| `password` | `password/passwd/pwd` + 8 chars | Password assignments |
| `token` | `token/secret` + 20 chars | Generic token assignments |
| `google_oauth_secret` | `client_secret` + 24 chars | OAuth client secrets |

## Overlap Merging

When multiple patterns match overlapping regions of text, the detector merges spans before redaction to prevent partial secret leakage:

```
Text:    "api_key=sk-ant-api03-very-long-key-here"
Match 1: [--------api_key match--------]
Match 2:         [---anthropic_key match---]
Merged:  [----------single span-----------]
Result:  "[REDACTED]"
```

Without merging, overlapping replacements could leave fragments of the secret visible.

## Usage

### Scanning

```python
from missy.security.secrets import secrets_detector

findings = secrets_detector.scan(text)
# Returns: [
#   {"type": "anthropic_key", "match_start": 42, "match_end": 89},
#   {"type": "api_key", "match_start": 42, "match_end": 85},
# ]
```

### Redaction

```python
safe_text = secrets_detector.redact(text)
# All detected secrets replaced with [REDACTED]
```

### Quick Check

```python
if secrets_detector.has_secrets(text):
    # Short-circuit: stops at first match
    handle_secret_detected()
```

### Module Singleton

The `secrets_detector` instance is a process-level singleton. Import and use directly:

```python
from missy.security.secrets import secrets_detector
```

## Integration Points

The secrets detector is used at multiple points in Missy's processing pipeline:

| Location | Purpose |
|---|---|
| Agent runtime | Scan tool outputs before passing to the model |
| CLI channel | Redact secrets from displayed responses |
| Audit logger | Prevent secrets in audit trail |
| Provider payloads | Scan outbound messages |
| Memory store | Prevent secret persistence in conversation history |

!!! warning "Pattern-based limitations"
    Secrets detection is regex-based and cannot catch every possible secret format. Custom or proprietary credential formats may not be detected. Use the [Vault](vault.md) to store secrets securely rather than relying solely on detection to prevent leakage.
