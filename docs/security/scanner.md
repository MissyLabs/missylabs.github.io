---
tags:
  - security
---

# Security Scanner

The `SecurityScanner` audits your Missy installation for common security issues. Run it with:

```bash
missy security scan
```

## What It Checks

| Category | Checks |
|----------|--------|
| **File permissions** | World-writable config files, exposed vault keys, loose directory permissions |
| **Config hygiene** | `default_deny` disabled, empty allowlists with shell enabled, missing config version |
| **Exposed secrets** | API keys in config files, tokens in environment, credentials in audit logs |
| **Dependencies** | Known CVEs in installed packages (when dependency data available) |

## Output

Findings are ranked by severity:

| Severity | Meaning |
|----------|---------|
| **Critical** | Immediate action required (e.g., vault key world-readable) |
| **High** | Significant risk (e.g., `default_deny: false`) |
| **Medium** | Recommended fix (e.g., config file group-writable) |
| **Low** | Best practice suggestion (e.g., missing config backup) |

## Example

```
$ missy security scan

Security Scan Results
━━━━━━━━━━━━━━━━━━━━
  HIGH   network.default_deny is false — all outbound traffic allowed
  MEDIUM ~/.missy/config.yaml is group-writable (0644)
  LOW    No config backups found — run 'missy config plan'

3 findings (0 critical, 1 high, 1 medium, 1 low)
```
