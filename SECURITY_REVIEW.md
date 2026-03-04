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

## Cross-Check with Official PyPI Package (v2.3.1)

**Official package:** https://pypi.org/project/linkedin-api/ (v2.3.1, Nov 2024)
**This repo version:** v1.1.0 (heavily outdated)

### This repo is a very old fork. The upstream has diverged significantly.

| Aspect | This Repo (v1.1.0) | PyPI Latest (v2.3.1) |
|--------|---------------------|----------------------|
| **SSL verify=False on auth** | YES — CRITICAL | NO — Fixed |
| **debug=True default** | YES — disables SSL globally | NO — defaults to `debug=False` |
| **disable_warnings()** | YES — silences SSL warnings | NO — Removed |
| **session.verify = not debug** | YES — SSL tied to debug flag | NO — Removed entirely |
| **Hardcoded credentials** | YES (test_acct.py) | NO |
| **Cookie storage** | Single `.cookie.jr` file via pickle | Per-user files in `~/.linkedin_api/cookies/` via pickle |
| **Cookie expiry check** | NO | YES — checks JSESSIONID expiry |
| **Python version** | 3.7+ | 3.10+ |
| **linkedin.py size** | 681 lines | ~1600+ lines (many new features) |
| **helpers.py** | 7 lines (1 function) | 267 lines (12+ functions) |
| **Auth user-agent** | iPhone 8.3 (2015-era) | Android (more current) |
| **Browser user-agent** | Chrome 66 (2018) | Chrome 83 (2020) |
| **Metadata fetching** | NO | YES — parses LinkedIn page for app instance data |
| **New file: cookie_repository.py** | NO | YES — dedicated cookie management class |
| **Mutable default args** | YES (`results=[]`) | NO — Fixed |
| **Features** | Basic (search, profile, messages) | Extended (posts, reactions, jobs, invitations, company pages) |

### Key Security Fixes in Upstream (v2.3.1)

1. **`verify=False` removed** — authentication uses default SSL verification (True)
2. **`disable_warnings()` removed** — security warnings are no longer silenced
3. **`session.verify = not debug` removed** — SSL is never disabled regardless of debug mode
4. **`debug=False` by default** — safe default
5. **Cookie expiry validation** — stale cookies raise `LinkedinSessionExpired`
6. **Per-user cookie files** — stored in `~/.linkedin_api/cookies/<username>.jr`

### Remaining Concerns in Upstream (v2.3.1)

1. **Cookies still stored via pickle** — unencrypted, but now per-user and in home directory
2. **No credential memory clearing** — passwords still in memory during session
3. **Pickle deserialization risk** — `pickle.load()` on cookie files (low risk since user-controlled)

### Verdict

**This repo is an outdated, insecure fork.** The upstream PyPI version (2.3.1) has fixed all three CRITICAL issues found in this repo. If you need this library, use the official PyPI package instead:
```
pip install linkedin-api
```

## Recommendations

1. **Remove `verify=False`** from authentication request (client.py:124)
2. **Change default `debug=False`** in Linkedin class (linkedin.py:36)
3. **Remove hardcoded credentials** from test_acct.py
4. **Remove `disable_warnings()`** call (client.py:56)
5. **Encrypt stored cookies** or use OS keychain instead of plaintext pickle
6. **Clear credentials from memory** after authentication
7. **Fix mutable default arguments** (`results=[]` → `results=None`)
8. **Or better yet: upgrade to upstream v2.3.1** which fixes issues 1-4 and 7
