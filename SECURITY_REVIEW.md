# Security Code Review: linkedin-api

**Date:** 2026-03-04
**Scope:** Full repository security audit
**Verdict:** Not malicious, but has significant security vulnerabilities

## Is it stealing information?

**No.** All HTTP requests target legitimate LinkedIn domains only:
- `https://www.linkedin.com/voyager/api` (API calls)
- `https://www.linkedin.com/uas/authenticate` (authentication)

No evidence found of:
- Data exfiltration to external servers
- Obfuscated code (base64, eval, exec)
- Backdoors, reverse shells, or keyloggers
- Webhook integrations (Telegram, Discord, Slack)
- Subprocess/os.system calls
- Email sending capabilities

## Security Issues Found

### CRITICAL

| # | Issue | Location |
|---|-------|----------|
| 1 | SSL verification disabled on auth POST (`verify=False`) — credentials vulnerable to MITM | `client.py:124` |
| 2 | Debug mode disables SSL globally (`session.verify = not debug`) and `Linkedin` class defaults to `debug=True` | `client.py:58`, `linkedin.py:36` |
| 3 | Hardcoded real credentials committed in plaintext | `test_acct.py:5` |

### HIGH

| # | Issue | Location |
|---|-------|----------|
| 4 | urllib3 security warnings silenced with `disable_warnings()` | `client.py:56` |
| 5 | Session cookies stored unencrypted via pickle (session hijacking + code execution risk) | `client.py:95-96` |
| 6 | Proxy parameter combined with disabled SSL allows full traffic interception | `client.py:59` |

### MEDIUM

| # | Issue | Location |
|---|-------|----------|
| 7 | Credentials remain in memory after authentication (no clearing) | `client.py:105` |
| 8 | Generic `raise Exception()` with no details on auth failure | `client.py:136` |
| 9 | Mutable default arguments in search/update methods cause state leakage | `linkedin.py:62,303,341` |

### LOW

| # | Issue | Location |
|---|-------|----------|
| 10 | Example code encourages storing credentials in plaintext JSON | `examples/basic.py` |
| 11 | Outdated user-agent strings (Chrome 66, iPhone 8.3) | `client.py:26-31,46` |

## Recommendations

1. **Remove `verify=False`** from authentication request (client.py:124)
2. **Change default `debug=False`** in Linkedin class (linkedin.py:36)
3. **Remove hardcoded credentials** from test_acct.py
4. **Remove `disable_warnings()`** call (client.py:56)
5. **Encrypt stored cookies** or use OS keychain instead of plaintext pickle
6. **Clear credentials from memory** after authentication
7. **Fix mutable default arguments** (`results=[]` → `results=None`)
