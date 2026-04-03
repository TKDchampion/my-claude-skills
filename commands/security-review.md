Perform a deep security review of the current codebase or specified files, covering OWASP, Trail of Bits, DB safety, concurrency, API abuse, and AI data leakage.

## Step 1: Define Scope

Ask if not specified:
```
Review scope:
  a = full project (all source files)
  d = git diff only (staged + unstaged changes)
  f = specific file(s) — provide paths
```
Wait for response, then gather the relevant code.

Also run in parallel:
- `git diff --cached && git diff` (if scope = d)
- `find . -name "*.py" -not -path "*/venv/*" -not -path "*/.git/*"` (if scope = a/f)

---

## Step 2: Static Analysis — Automated Tools

Run available tools automatically (skip silently if not installed):

### Python
```bash
# Bandit — OWASP / CWE static analysis
bandit -r . -ll -ii --exclude ./venv,./tests -f text

# Safety — known CVEs in dependencies
safety check --full-report

# Semgrep — Trail of Bits + community rules
semgrep --config=p/python --config=p/owasp-top-ten --config=p/trailofbits --config=p/secrets --quiet .
```

### Node / TypeScript
```bash
npm audit --audit-level=moderate
semgrep --config=p/javascript --config=p/owasp-top-ten --config=p/trailofbits --quiet .
```

### Universal
```bash
# Gitleaks — secrets / credentials in code and history
gitleaks detect --source . --no-git -v

# Trivy — dependency CVE scan
trivy fs . --severity HIGH,CRITICAL --quiet
```

Collect all findings. If no tools installed → proceed with manual analysis only and note "Static tools not available — manual review only."

---

## Step 3: OWASP Top 10 (2025) Manual Review

Scan source code for each category:

| # | Category | What to look for |
|---|----------|-----------------|
| A01 | Broken Access Control | Missing auth checks, IDOR patterns, path traversal, forced browsing |
| A02 | Cryptographic Failures | Plaintext secrets, weak algorithms (MD5/SHA1/DES), hardcoded keys, unencrypted PII |
| A03 | Injection | SQLi via f-string/concatenation, command injection, SSTI, LDAP injection |
| A04 | Insecure Design | Business logic flaws, missing threat model, lack of rate limiting |
| A05 | Security Misconfiguration | Debug mode in prod, permissive CORS, default credentials, verbose errors |
| A06 | Vulnerable Components | Outdated deps, known CVE packages, unmaintained libraries |
| A07 | Auth & Session Failures | Weak JWT (alg:none, no expiry), session fixation, missing MFA paths |
| A08 | Software Integrity | Unverified downloads, unsigned packages, CI/CD pipeline injection |
| A09 | Logging Failures | Sensitive data in logs, missing audit trail, no anomaly detection |
| A10 | SSRF | User-controlled URLs fetched server-side, missing allowlists |

---

## Step 4: Trail of Bits Deep Checks

Focus on logic-level vulnerabilities often missed by scanners:

- **Integer overflow / underflow** in arithmetic operations
- **Type confusion** — unchecked type coercion, unsafe casts
- **Reentrancy patterns** — callback-based logic that can be interrupted
- **Incorrect error handling** — swallowed exceptions, silent failures that break security invariants
- **Insecure randomness** — `random` instead of `secrets`/`os.urandom` for security-sensitive values
- **Prototype pollution** (JS) — recursive merge on untrusted objects
- **Denial-of-service via input** — unbounded loops, regex backtracking (ReDoS), large payload allocation
- **Trust boundary violations** — internal service calls that skip auth assuming internal network safety

---

## Step 5: Database Security & Performance

### Security
- Raw string interpolation in queries → SQLi risk
- ORM misuse: `.execute()` with user input, `.filter()` with `text()`
- Mass assignment via `**kwargs` / `**request.dict()` without field allowlist
- Missing row-level security or tenant isolation in multi-tenant queries
- Sensitive columns returned without field filtering (password hash, tokens in SELECT *)
- Transactions not used where atomicity is required (partial write risk)

### Performance Indicators
- **N+1 query patterns** — loop containing a DB call (flag as `[PERF-N+1]`)
- **Missing indexes** implied by filter/order columns — cross-reference entity definitions
- **Unbounded queries** — no `.limit()` on list endpoints
- **SELECT * on large tables** — unnecessary column fetch
- **Missing connection pool configuration** — check database.py settings
- **Long-running transactions** — DB locks held across external API calls

---

## Step 6: Race Conditions & Transaction Safety

- **TOCTOU (Time-of-check/Time-of-use)** — check then act patterns without atomic DB lock
- **Non-atomic read-modify-write** — increment/decrement without `SELECT FOR UPDATE` or atomic update
- **Missing transaction boundaries** — multi-step operations that should be wrapped in `@db_tx`
- **Concurrent request state sharing** — mutable module-level or class-level state
- **Optimistic locking** — version fields present but not enforced on update
- **Deadlock risk** — multiple resources locked in inconsistent order

Flag patterns as `[RACE]` with the file and line.

---

## Step 7: API Abuse Vectors

- **Rate limiting** — endpoints missing throttle decorators or middleware; check login, OTP, search, export endpoints especially
- **Replay attacks** — JWT without `jti` claim, webhook signatures not checked for timestamp freshness (>5 min tolerance)
- **Mass enumeration** — sequential IDs exposed, predictable resource paths, verbose 404 vs 403 distinction leaking existence
- **Excessive data exposure** — response includes fields the caller's role should not see
- **GraphQL / batch endpoint abuse** — unbounded query depth, no query cost limit
- **BOLA (Broken Object Level Auth)** — resource ID in path/body not verified against caller's ownership
- **Parameter pollution** — duplicate query params, array injection in filters

---

## Step 8: AI / LLM API Data Leakage

Relevant when code calls OpenAI, Anthropic, Gemini, WrenAI, or any LLM endpoint:

- **PII in prompts** — user data (email, name, ID) injected into prompt without sanitization
- **System prompt exposure** — system prompt reconstructible from API response or error
- **Prompt injection** — user-controlled text inserted directly into prompt without delimiter/escaping
- **Full conversation history sent** — entire thread sent to LLM including previous users' messages
- **API key leakage** — key logged, returned in response, or stored in DB alongside output
- **Model response stored unfiltered** — LLM output written to DB/file without content validation
- **Streaming response leaks** — SSE/streaming endpoint exposes chunks to unauthorized callers
- **Token budget abuse** — no max_tokens limit, attacker can force expensive completions

Flag patterns as `[AI-LEAK]`.

---

## Step 9: Present Full Report

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  DEEP SECURITY REVIEW REPORT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

🛠  Static Tools
  <bandit / semgrep / safety / gitleaks findings or "not available">

🔐 OWASP Top 10
  [A0X] <file>:<line> — <description>
  ...
  (or ✓ No issues found)

🔬 Trail of Bits
  <file>:<line> — <description>
  ...

🗄  Database
  [SQLi / PERF-N+1 / TX / etc.] <file>:<line> — <description>
  ...

⚡ Race Conditions
  [RACE] <file>:<line> — <description>
  ...

🚦 API Abuse
  <finding>
  ...

🤖 AI / LLM Leakage
  [AI-LEAK] <file>:<line> — <description>
  ...

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  SEVERITY SUMMARY
  🔴 Critical: X   🟠 High: X   🟡 Medium: X   🟢 Low: X

  OVERALL: <SECURE / REVIEW RECOMMENDED / ACTION REQUIRED>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Top Priority Fixes:
  1. [Critical] <action>
  2. [High]     <action>
  3. [Medium]   <action>
```

---

## Step 10: Ask for Next Action

```
What would you like to do?

  f = auto-fix Critical and High issues now
  s = show fix suggestions only (no changes)
  e = export report to security-report.md
  c = cancel
```

**Wait for response before taking any action.**

- `f` → Apply fixes for Critical/High findings, then re-run Steps 3–8 to confirm resolution
- `s` → Show detailed code-level fix for each finding without modifying files
- `e` → Write full report to `security-report.md` in project root
- `c` → "Cancelled. No changes made."

**Never auto-fix without confirmation.**
