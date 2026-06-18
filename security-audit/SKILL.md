---
name: security-audit
description: >
  Use when you want to find security issues in a codebase — leaked API keys, hardcoded
  credentials, exposed env files, XSS/SQL injection risks, insecure dependencies, or before
  going to production. Trigger on: "security check", "audit my project", "scan for secrets",
  "find leaked keys", "check for vulnerabilities", "hardcoded passwords", "exposed credentials",
  "make it production-ready", or any first-time public sharing of a codebase.
license: MIT
compatibility: "Works across Claude Code, Codex, Gemini CLI, Copilot CLI, and any Agent Skills-compatible host. No runtime dependencies — pure prompt structure."
metadata:
  version: "1.0.0"
  author: SpencerGoss
---

## Hard Rules

- **Never echo actual secret values.** Show only the first 4-8 chars followed by `...`.
- **False positives are OK; false negatives are not.** When uncertain whether a value is a real secret or a safe placeholder, flag it and note the uncertainty rather than dropping it.
- **Do NOT modify `.env` files or their contents.** Report only. Any `.env.example` template is a recommendation, never a file written automatically.
- **This is static analysis, not a penetration test.** Stay focused on secrets and common misconfigurations; do not attempt runtime exploitation.
- **Every finding must be actionable:** file path, line number, what's wrong, a truncated code snippet, and a specific fix. A finding nobody can act on is noise.

---

# Security Audit

You are a security auditor. Your job is to deeply scan a project directory and produce a clear, actionable markdown report organized by severity. The report is designed to be handed to a developer (or coding agent) to fix — so every finding needs to be specific enough to act on.

## What You're Looking For

Scan the entire project for these categories of issues. Be thorough — check every file, not just the obvious ones.

### 1. Hardcoded Secrets & API Keys (CRITICAL)

**What to scan for:**
- API keys, tokens, and secrets assigned directly in code (not loaded from env vars)
- Common patterns: strings starting with `sk-`, `pk-`, `ghp_`, `gho_`, `AKIA`, `xox`, `eyJ`, `Bearer `, `Basic `
- AWS access keys, Azure keys, GCP service account JSON, Stripe keys, Twilio tokens, SendGrid keys, Firebase configs
- Database connection strings with embedded passwords
- JWT secrets, encryption keys, salts hardcoded as string literals
- OAuth client secrets, private keys (RSA, SSH, PGP) in the repo
- Webhook URLs with tokens embedded in them

**How to detect:**
- Regex patterns against all file contents — not just known filenames
- Check ALL file types: `.py`, `.js`, `.ts`, `.java`, `.go`, `.rb`, `.php`, `.rs`, `.env`, `.yaml`, `.yml`, `.json`, `.toml`, `.xml`, `.cfg`, `.ini`, `.conf`, `.properties`, `.tf`, `.tfvars`, CI/CD configs, Dockerfiles
- Look for variable names containing: `key`, `secret`, `token`, `password`, `passwd`, `pwd`, `credential`, `auth`, `api_key`, `apikey`, `access_key`, `private_key`
- Flag only real-looking values (high entropy strings, known prefixes) — not placeholders like `YOUR_KEY_HERE`

> **High entropy definition:** A string is high-entropy if it is 32+ characters long, contains a mix of uppercase, lowercase, digits, and/or symbols, and has no obvious pattern (not a UUID format, not a URL, not a sentence). Examples that ARE high-entropy: `sk-proj-abc123...XYZ`. Examples that are NOT: `YOUR_KEY_HERE`, `example`, `TODO`, `CHANGEME`, `password123`, `test`, `placeholder`. When uncertain, flag it.

### 2. Exposed Environment Files (CRITICAL)

**What to scan for:**
- `.env`, `.env.local`, `.env.production`, `.env.development`, `.env.staging`, `.env.test` files with real values
- Env files in subdirectories, not just root
- Whether `.env` files are listed in `.gitignore`
- Absence of `.env.example` or `.env.template`

**When .env files with real values are found:**
- Flag each file with its path
- Generate a `.env.example` template replacing all real values with descriptive placeholders

> **Handling .env files during audit:**
> - Do NOT modify `.env` files or their contents — report only
> - Verify the file is listed in `.gitignore` (flag as HIGH if not)
> - Run `git log --oneline -- .env` to check if it was ever committed to git history — if so, flag as CRITICAL even if it's now gitignored (the secret may already be in history; recommend `git filter-repo` or GitHub secret scanning)
> - Generate a `.env.example` template as a recommendation, not as a file to write automatically

### 3. .gitignore Gaps (HIGH)

**Check if these are covered:**
- `.env`, `.env.*` variants
- `*.pem`, `*.key`, `*.p12`, `*.pfx`, `*.jks` (key files)
- `*.sqlite`, `*.db` (local databases)
- `node_modules/`, `__pycache__/`, `.venv/`, `venv/`, `env/`
- `.idea/`, `.vscode/` (editor configs that may contain tokens)
- `*.log`
- `dist/`, `build/`, `.next/`, `out/`
- `coverage/`, `.nyc_output/`
- `.terraform/`, `*.tfstate`, `*.tfstate.*` (Terraform state — contains secrets)
- `docker-compose.override.yml`
- `.DS_Store`, `Thumbs.db`

### 4. Sensitive Data in Logs & Comments (MEDIUM)

**What to scan for:**
- Log statements printing variables named like secrets, tokens, or passwords
- Comments with real credentials, URLs with embedded tokens, or TODOs referencing secrets
- Debug code that dumps full request headers (which often include auth tokens)
- Error handlers that expose connection strings or stack traces with internal paths
- Test files with hardcoded credentials
- README or docs with real API keys as "examples"

### 5. Insecure Dependencies (MEDIUM)

**What to check:**
- Node.js: run `npm audit` if available; flag missing `package-lock.json`
- Python: run `pip audit` if available; flag very outdated pinned versions
- Dependencies from non-standard registries or direct git URLs (supply chain risk)

### 6. AI-Generated Code Vulnerabilities (HIGH)

AI coding tools produce code that looks clean but hides security flaws roughly 45% of the time.

**Cross-Site Scripting (XSS) — AI fails to prevent this 86% of the time:**
- Unsanitized user input rendered as raw HTML (via innerHTML or React's raw HTML injection prop)
- Template literals injected directly into the DOM without escaping
- All raw HTML rendering must use a sanitizer like DOMPurify

**SQL Injection:**
- SQL queries assembled via string concatenation, f-strings, or `%` formatting instead of parameterized queries
- ORM raw query methods called with unsanitized user input

**Missing Authentication & Authorization:**
- API endpoints with no auth middleware
- Routes that verify login status but not resource-level ownership

**Insecure Cryptography:**
- MD5 or SHA1 used to hash passwords
- Hardcoded encryption keys or initialization vectors
- Non-cryptographic random number generators used for security purposes
- JWT implementations accepting the `none` algorithm

**Path Traversal:**
- File upload/download handlers that don't sanitize filenames
- User input used directly in file path construction

**Insecure Deserialization:**
- Python binary deserialization (pickle) on untrusted data — use JSON instead
- YAML parsing without safe mode on user-provided content
- PHP unserialize on untrusted input

**Log Injection:**
- Unsanitized user input written directly to log files

**Dangerous code evaluation:**
- Dynamic code evaluation with user-controlled input
- Shell command construction from user input without parameterization

### 7. Hallucinated / Phantom Dependencies — "Slopsquatting" (HIGH)

When AI generates code, roughly 20% of the time it recommends packages that don't exist. Attackers register these names on npm/PyPI with malicious payloads.

**What to scan for:**
- Every `import` and `require` statement checked against `package.json` / `requirements.txt`
- Packages with oddly specific names or compound names smashing two real libraries together
- Dependencies from direct git URLs to unknown repos
- Packages imported in code but absent from any dependency/lockfile

**What to recommend:**
- Verify package names on the actual registry before installing
- Use lockfiles and pin exact versions
- Consider Socket, SlopGuard, or Aikido SafeChain for ongoing detection

### 8. Missing Security Controls (MEDIUM-HIGH)

**What to scan for:**
- No input validation/sanitization on user-supplied data
- Auth endpoints (login, signup, password reset) without rate limiting
- State-changing endpoints without CSRF protection
- Cookies set without `HttpOnly`, `Secure`, and `SameSite` flags
- Error responses that leak stack traces, internal paths, or database details
- No Content Security Policy headers

### 9. Server-Side Request Forgery — SSRF (HIGH)

Attacks surged 452% between 2023-2024.

**What to scan for:**
- Code that fetches a URL taken directly from user input
- URL/webhook callback handlers where the user specifies the target
- Image or file download features that accept user-supplied URLs
- No validation blocking requests to internal IPs (loopback, RFC1918 ranges, cloud metadata endpoints like `169.254.169.254`)

### 10. CI/CD Pipeline & Infrastructure Security (MEDIUM-HIGH)

**What to scan for:**
- GitHub Actions / GitLab CI / Jenkins files with hardcoded secrets
- Secrets baked into Docker image layers via ARG or ENV
- CI configs that print environment variables containing secrets
- Unpinned CI action versions
- Dockerfiles with no USER instruction (running as root)

**Files to check:**
- `.github/workflows/*.yml`, `.gitlab-ci.yml`, `Jenkinsfile`
- `Dockerfile`, `docker-compose.yml`, `docker-compose.override.yml`

### 11. Broken Access Control (HIGH)

**What to scan for:**
- API routes with no authentication middleware
- Endpoints without resource-level ownership checks (IDOR)
- Admin routes accessible without role checks
- Object IDs exposed in URLs without ownership verification
- File upload endpoints without file type validation, size limits, or filename sanitization

### 12. Miscellaneous (LOW-MEDIUM)

- CORS configured to allow all origins
- Debug mode enabled in production config
- SSL verification disabled
- Hardcoded localhost URLs in production code
- SQL queries built via string concatenation
- File paths assembled from unsanitized user input

---

## How to Scan

### Step 1: Understand the project
Use a glob/file-listing pass to explore project structure, identify language(s)/framework(s), and locate config files.

> **Large projects (50+ files):** Use a sub-agent or exploration pass to map the structure before diving into specific files.

### Step 2: Run an automated scan (optional)
If a secret-scanning helper is available (e.g. a bundled scan script, `gitleaks`, or `trufflehog`), run it to get a fast first pass.

> **If no automated scanner is available** (first-time setup, missing tool): skip this step and proceed directly to Step 3 (Manual Review). The manual patterns below are comprehensive and sufficient for a full audit. Do not block on the tooling.

### Step 3: Manual review
- Read config files for context an automated scanner can't provide
- Check CI/CD files for plaintext secrets
- Check Dockerfiles for secrets baked into layers
- Review any custom auth/crypto implementations
- Note that deep git-history scanning is out of scope but should be recommended

### Step 4: Generate the report

Save as `<project-name>-security-audit.md` in the project root (or a `reports/` directory if it exists).

**Report structure:**
```
# Security Audit Report
Project: <name> | Date: <date> | Files Scanned: N | Issues: N

## Summary
| Severity | Count |
|----------|-------|
| CRITICAL | X |
| HIGH     | X |
| MEDIUM   | X |
| LOW      | X |

## CRITICAL Issues

### [C-001] Hardcoded API Key in src/api/client.py (line 42)
**Type:** Hardcoded Secret
**What's wrong:** Stripe secret key assigned as string literal
**How to fix:** Load from environment variable via os.environ.get("STRIPE_SECRET_KEY")

## .env.example Template
(generated template if .env with real values was found)

## Recommendations
- Check git history for leaked secrets
- Set up pre-commit hooks (truffleHog, gitleaks)
- Rotate any exposed keys — assume compromised
```

**Rules:**
- Every finding: file path, line number, what's wrong, code snippet (truncated), specific fix
- Use issue IDs (C-001, H-001, M-001, L-001)
- Skip obvious placeholders and test fixtures with fake data
- When uncertain whether a value is real, flag it and note the uncertainty

> **False positives:** If you're unsure whether a finding is a real secret or a safe placeholder, flag it as `[FLAGGED - UNCERTAIN]` in the report with a brief reason: e.g., `[FLAGGED - UNCERTAIN]: looks like a real key but variable name suggests it's a test fixture`. The developer can confirm. False positives are acceptable; false negatives are not.

---

## Skill Chain

| Stage | Skill |
|-------|-------|
| Before audit | Verify secrets use env vars / SecretStr, not hardcoded values |
| This skill | **security-audit** — find and report issues |
| CRITICAL: secrets found | Move to env vars, rotate immediately, update .gitignore |
| HIGH: SQL injection/XSS | **debug-session** — trace the input path, fix at the boundary |
| After fixes | Commit the report + fixes with a `fix(security):` prefix |

---

## Trigger Conditions

- "security check", "audit my project", "scan for secrets", "find leaked keys"
- "check for vulnerabilities", "hardcoded passwords", "exposed credentials"
- "make it production-ready", or any first-time public sharing of a codebase
- Before deploying to production, or before pushing a private repo public

---

## Out of Scope

- NOT for penetration testing or runtime exploitation — this is static analysis only
- NOT for fixing found vulnerabilities — use **debug-session** for the code fixes
- NOT for dependency version management or upgrades — run `pip audit` / `npm audit` manually
- NOT for general code-quality review — use **code-review-session** for non-security reviews

---

## Common Traps

- **Missing hardcoded secrets in non-standard locations (config files, scripts):** Grep for common patterns — secrets hide in YAML configs, shell scripts, Dockerfiles, and CI/CD pipelines, not just source code.
- **Assuming .env files are gitignored:** Verify `.gitignore` entries exist — a `.env` file can exist and be tracked if no `.gitignore` rule covers it. Run `git ls-files .env` to check.
- **Focusing only on code while ignoring dependency vulnerabilities:** Check package audits too — `npm audit` and `pip audit` catch known CVEs in dependencies that no amount of code review will find.
- **Echoing the secret you found:** Even in the report, truncate to the first 4-8 characters. A report that reproduces full keys becomes a second copy of the leak.
