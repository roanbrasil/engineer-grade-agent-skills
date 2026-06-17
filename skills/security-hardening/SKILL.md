---
name: security-hardening
description: Production security hardening — OWASP Top 10, secret management, JWT security, input validation, security headers, threat modeling, SAST, and container hardening.
---

# Security Hardening — Expert Reference

## OWASP Top 10 (2021)

### 1. Broken Access Control

**What it is:** Authenticated users can access resources or actions they are not authorized for. The most common OWASP finding.

**Detection:**
- Users can access other users' data by changing an ID in the URL: `/api/orders/12345` → `/api/orders/12346`
- Horizontal privilege escalation: regular user accessing admin endpoints
- Missing authorization on server-side even though UI hides the button

**Prevention:**

```python
# BAD: trust the user-supplied ID
@app.get("/api/orders/{order_id}")
def get_order(order_id: int, current_user: User = Depends(get_current_user)):
    return db.query(Order).filter(Order.id == order_id).first()

# GOOD: scope the query to the authenticated user
@app.get("/api/orders/{order_id}")
def get_order(order_id: int, current_user: User = Depends(get_current_user)):
    order = (
        db.query(Order)
        .filter(Order.id == order_id, Order.user_id == current_user.id)
        .first()
    )
    if not order:
        raise HTTPException(status_code=404)  # 404 not 403 — don't confirm existence
    return order
```

```java
// Spring Security — method-level authorization
@PreAuthorize("@orderSecurityService.canAccess(authentication, #orderId)")
@GetMapping("/api/orders/{orderId}")
public ResponseEntity<Order> getOrder(@PathVariable Long orderId) { ... }
```

**Rule:** Authorize at every layer (controller, service, data). Never rely on UI hiding.

---

### 2. Cryptographic Failures

**What it is:** Sensitive data exposed due to weak or absent encryption.

**Common failures:**
- Passwords stored as MD5/SHA1 or unsalted SHA256
- HTTP instead of HTTPS (plaintext in transit)
- Weak TLS cipher suites
- Encryption keys hardcoded or stored alongside encrypted data

**Prevention:**

```python
# BAD: MD5 password hashing
import hashlib
hashed = hashlib.md5(password.encode()).hexdigest()  # crackable instantly

# BAD: SHA256 without salt
hashed = hashlib.sha256(password.encode()).hexdigest()  # rainbow table attack

# GOOD: bcrypt (Python)
import bcrypt
hashed = bcrypt.hashpw(password.encode(), bcrypt.gensalt(rounds=12))
is_valid = bcrypt.checkpw(password.encode(), hashed)

# GOOD: argon2 (recommended over bcrypt for new systems)
from argon2 import PasswordHasher
ph = PasswordHasher(time_cost=2, memory_cost=65536, parallelism=2)
hashed = ph.hash(password)
is_valid = ph.verify(hashed, password)
```

```java
// Spring Security — BCrypt
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder(12);  // cost factor 12 = ~300ms on modern CPU
}
```

**TLS checklist:**
```
[ ] TLS 1.2 minimum; TLS 1.3 preferred
[ ] Disable SSLv3, TLS 1.0, TLS 1.1
[ ] Strong cipher suites only (ECDHE, AES-GCM, CHACHA20)
[ ] HSTS header enabled (Strict-Transport-Security)
[ ] Certificate rotation automated (Let's Encrypt / ACM)
[ ] No self-signed certs in production
```

---

### 3. Injection (SQL, NoSQL, OS Command, LDAP)

**What it is:** Untrusted data is sent to an interpreter as part of a command or query.

**SQL Injection — Prevention:**

```python
# BAD: string concatenation
query = f"SELECT * FROM users WHERE email = '{email}'"
db.execute(query)

# GOOD: parameterized query (SQLAlchemy)
user = db.execute(
    text("SELECT * FROM users WHERE email = :email"),
    {"email": email}
).first()

# GOOD: ORM (SQLAlchemy)
user = db.query(User).filter(User.email == email).first()
```

```java
// BAD: string concatenation
String query = "SELECT * FROM users WHERE email = '" + email + "'";
stmt.execute(query);

// GOOD: PreparedStatement
PreparedStatement stmt = conn.prepareStatement(
    "SELECT * FROM users WHERE email = ?"
);
stmt.setString(1, email);
ResultSet rs = stmt.executeQuery();

// GOOD: Spring Data JPA
Optional<User> findByEmail(String email);  // parameterized automatically
```

**OS Command Injection:**

```python
# BAD: shell=True with user input
import subprocess
subprocess.run(f"convert {filename} output.png", shell=True)
# filename = "input.jpg; rm -rf /"

# GOOD: list form, no shell
subprocess.run(["convert", filename, "output.png"])
# Each argument is passed directly, no shell interpretation
```

**NoSQL Injection (MongoDB):**

```javascript
// BAD: user controls query operator
const user = await db.collection('users').findOne({
    username: req.body.username,
    password: req.body.password  // can be { $gt: "" }
});

// GOOD: validate that inputs are strings, not objects
const { username, password } = req.body;
if (typeof username !== 'string' || typeof password !== 'string') {
    return res.status(400).json({ error: 'Invalid input' });
}
```

---

### 4. Insecure Design

**What it is:** Missing or ineffective security controls at the design level. Cannot be fixed by patching — requires redesign.

**Prevention — Threat Modeling (STRIDE):**

```
For each component and data flow, ask:
  S — Spoofing: can an attacker impersonate a user or service?
  T — Tampering: can data be modified in transit or at rest?
  R — Repudiation: can an actor deny performing an action?
  I — Information Disclosure: can data be exposed to unauthorized parties?
  D — Denial of Service: can the component be made unavailable?
  E — Elevation of Privilege: can a low-privilege user gain higher access?

For each threat:
  1. Is it a real threat given our architecture?
  2. What is the impact (High/Medium/Low)?
  3. What is the likelihood?
  4. Mitigation: accept, mitigate, or transfer (insurance)?
```

**Example threat model entry:**

```
Component: Password Reset Flow
Threat (STRIDE: S): Attacker uses expired/previously-used reset token
Impact: High (account takeover)
Mitigation:
  - Tokens expire after 15 minutes
  - Tokens are single-use (invalidated after first use)
  - Tokens are stored as HMAC, not plaintext
  - Rate-limit reset requests per email address (5/hour)
```

---

### 5. Security Misconfiguration

**What it is:** Insecure default settings, verbose errors in production, unnecessary features enabled.

**Checklist:**

```
[ ] Default credentials changed (database, admin panels, cloud consoles)
[ ] Stack traces and internal errors not exposed in API responses
[ ] Debug endpoints disabled in production (/actuator/*, /debug/*)
[ ] Directory listing disabled on web server
[ ] Unnecessary HTTP methods disabled (TRACE, OPTIONS where not needed)
[ ] Cloud storage buckets not publicly accessible
[ ] Firewall rules: least privilege (deny by default, allow specific ports)
[ ] Secrets not in application.properties / application.yml / .env files committed to git
```

```python
# BAD: verbose error in production
@app.exception_handler(Exception)
async def exception_handler(request, exc):
    return JSONResponse({"error": str(exc), "traceback": traceback.format_exc()})

# GOOD: log internally, return generic message
@app.exception_handler(Exception)
async def exception_handler(request, exc):
    logger.exception("Unhandled exception", exc_info=exc)
    return JSONResponse({"error": "An internal error occurred"}, status_code=500)
```

---

### 6. Vulnerable and Outdated Components

**What it is:** Running libraries or frameworks with known CVEs.

**Prevention:**

```bash
# Python
pip audit                          # official PyPI advisory check
pip-audit --fix                    # auto-upgrade where safe
safety check                       # alternative scanner

# Node.js
npm audit
npm audit fix

# Java (Maven)
mvn org.owasp:dependency-check-maven:check

# Rust
cargo audit

# Docker image scanning
trivy image myapp:latest
grype myapp:latest
```

**Automation:**

```yaml
# GitHub Dependabot (.github/dependabot.yml)
version: 2
updates:
  - package-ecosystem: "pip"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
    target-branch: "main"
    labels: ["security", "dependencies"]

  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "weekly"
```

**SBOM (Software Bill of Materials):**

```bash
# Generate SBOM with Syft
syft myapp:latest -o spdx-json > sbom.json

# Scan SBOM for vulnerabilities
grype sbom:sbom.json
```

---

### 7. Identification and Authentication Failures

**What it is:** Weak authentication, credential stuffing, session management failures.

**Prevention:**

```python
# MFA — TOTP (Time-based One-Time Password)
import pyotp

# Enrolment: generate secret per user, show QR code
secret = pyotp.random_base32()
totp = pyotp.TOTP(secret)
qr_uri = totp.provisioning_uri(user.email, issuer_name="MyApp")

# Verification
totp = pyotp.TOTP(user.totp_secret)
if not totp.verify(user_provided_code, valid_window=1):
    raise AuthenticationError("Invalid MFA code")
```

```python
# Rate limiting + account lockout (example with Redis)
LOCKOUT_THRESHOLD = 5
LOCKOUT_DURATION = 900  # 15 minutes

def check_lockout(username: str) -> None:
    key = f"login_attempts:{username}"
    attempts = redis.get(key)
    if attempts and int(attempts) >= LOCKOUT_THRESHOLD:
        raise AccountLockedError("Account temporarily locked")

def record_failed_attempt(username: str) -> None:
    key = f"login_attempts:{username}"
    pipe = redis.pipeline()
    pipe.incr(key)
    pipe.expire(key, LOCKOUT_DURATION)
    pipe.execute()

def clear_attempts(username: str) -> None:
    redis.delete(f"login_attempts:{username}")
```

**Session security checklist:**
```
[ ] Session ID: cryptographically random, >= 128 bits
[ ] Rotate session ID after login (prevent session fixation)
[ ] Invalidate session on logout (server-side)
[ ] Absolute session timeout: 8 hours (even with activity)
[ ] Idle session timeout: 30 minutes
[ ] HttpOnly cookie: prevents XSS access to session cookie
[ ] Secure cookie: HTTPS only
[ ] SameSite=Strict or Lax: prevents CSRF
```

---

### 8. Software and Data Integrity Failures

**What it is:** Code or data from untrusted sources without integrity verification. Supply chain attacks.

**Prevention:**

```bash
# Verify artifact signatures
cosign verify ghcr.io/myorg/myapp:latest  # Sigstore/cosign

# Pin dependencies to exact versions (not ranges)
# Python: pin in requirements.txt
cryptography==42.0.5  # not cryptography>=42

# Node.js: use package-lock.json (already exact), verify integrity
npm ci  # uses lockfile exactly, fails if package.json differs

# Docker: pin image digest, not tag
FROM python:3.12.3-slim@sha256:abc123...  # not python:3.12-slim
```

**GitHub Actions — supply chain:**

```yaml
# BAD: use a mutable tag
- uses: actions/checkout@v4

# GOOD: pin to commit SHA
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
```

---

### 9. Security Logging and Monitoring Failures

**What it is:** Insufficient logging to detect and respond to breaches.

**What to log:**

```python
# GOOD: security-relevant events to log
SECURITY_EVENTS = [
    "user_login_success",
    "user_login_failure",
    "user_logout",
    "password_changed",
    "mfa_enabled",
    "mfa_disabled",
    "password_reset_requested",
    "account_locked",
    "permission_denied",
    "admin_action",
    "data_export",
    "api_key_created",
    "api_key_revoked",
]

import structlog

log = structlog.get_logger()

def log_security_event(event: str, user_id: int, request, **kwargs):
    log.info(
        event,
        user_id=user_id,
        ip_address=request.client.host,
        user_agent=request.headers.get("user-agent"),
        timestamp=datetime.utcnow().isoformat(),
        **kwargs
    )
```

**What NOT to log:**

```
NEVER log:
  - Passwords (plaintext or otherwise)
  - Credit card numbers (even last 4 if not necessary)
  - API keys or tokens (log the key ID, not the value)
  - Session tokens
  - Social Security Numbers / National IDs
  - Full credit card numbers
  - Encryption keys
  - Health data beyond what the feature requires
```

**Structured logging for SIEM:**

```json
{
  "timestamp": "2024-01-15T14:32:11Z",
  "event": "user_login_failure",
  "user_id": null,
  "email_hash": "sha256:abc123",
  "ip_address": "203.0.113.42",
  "user_agent": "Mozilla/5.0...",
  "reason": "invalid_password",
  "attempt_count": 3
}
```

---

### 10. Server-Side Request Forgery (SSRF)

**What it is:** Application fetches a URL supplied by the user. Attacker uses it to reach internal services (AWS metadata, databases, internal APIs).

**Prevention:**

```python
# BAD: fetch any URL the user provides
import requests

@app.post("/api/webhook-test")
def test_webhook(url: str):
    response = requests.get(url)  # can hit http://169.254.169.254/metadata
    return response.json()

# GOOD: validate URL before fetching
import ipaddress
from urllib.parse import urlparse

ALLOWED_SCHEMES = {"https"}
BLOCKED_NETWORKS = [
    ipaddress.ip_network("10.0.0.0/8"),
    ipaddress.ip_network("172.16.0.0/12"),
    ipaddress.ip_network("192.168.0.0/16"),
    ipaddress.ip_network("169.254.0.0/16"),  # link-local / AWS metadata
    ipaddress.ip_network("127.0.0.0/8"),     # loopback
]

def validate_external_url(url: str) -> None:
    parsed = urlparse(url)
    if parsed.scheme not in ALLOWED_SCHEMES:
        raise ValueError(f"Scheme not allowed: {parsed.scheme}")

    import socket
    ip = socket.gethostbyname(parsed.hostname)
    ip_obj = ipaddress.ip_address(ip)

    for network in BLOCKED_NETWORKS:
        if ip_obj in network:
            raise ValueError(f"Internal addresses not allowed: {ip}")
```

---

## Secret Management

### The Rules

```
1. Secrets never enter source code. Not even for one commit. Git history is permanent.
2. Secrets are not in environment variables of the system's .env files committed to repo.
3. Secrets live in a secrets manager (Vault, AWS Secrets Manager, GCP Secret Manager).
4. Each environment has its own secrets. Dev != Staging != Prod.
5. Rotate secrets regularly. Rotation must not require a deploy.
6. Least privilege: each service gets only the secrets it needs.
```

### Secret Detection — Pre-commit Hook

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks

  - repo: https://github.com/trufflesecurity/trufflehog
    rev: v3.63.0
    hooks:
      - id: trufflehog
        entry: trufflehog git file://. --since-commit HEAD --only-verified --fail
```

```bash
# Scan git history for secrets
gitleaks detect --source . --log-opts HEAD~50..HEAD
trufflehog git https://github.com/myorg/myrepo --only-verified
```

### AWS Secrets Manager Pattern

```python
import boto3
import json
from functools import cache

@cache
def get_secret(secret_name: str) -> dict:
    client = boto3.client("secretsmanager", region_name="us-east-1")
    response = client.get_secret_value(SecretId=secret_name)
    return json.loads(response["SecretString"])

# Usage
db_creds = get_secret("prod/myapp/database")
db_url = f"postgresql://{db_creds['username']}:{db_creds['password']}@{db_creds['host']}/mydb"
```

### HashiCorp Vault

```python
import hvac

client = hvac.Client(url="https://vault.internal:8200", token=os.environ["VAULT_TOKEN"])

# Read secret
secret = client.secrets.kv.v2.read_secret_version(
    path="myapp/database",
    mount_point="secret"
)
db_password = secret["data"]["data"]["password"]
```

---

## JWT Security

```python
# BAD: HS256 with shared secret in multi-service setup
token = jwt.encode(payload, "shared_secret_everyone_knows", algorithm="HS256")

# GOOD: RS256 — each service validates with public key, only auth service has private key
import jwt
from cryptography.hazmat.primitives import serialization

# Auth service: sign with private key
with open("private.pem", "rb") as f:
    private_key = serialization.load_pem_private_key(f.read(), password=None)

token = jwt.encode(payload, private_key, algorithm="RS256")

# Other services: validate with public key (no signing capability)
with open("public.pem", "rb") as f:
    public_key = serialization.load_pem_public_key(f.read())

payload = jwt.decode(
    token,
    public_key,
    algorithms=["RS256"],
    audience="payment-service",        # validate aud
    issuer="auth.myapp.com",           # validate iss
    options={"require": ["exp", "iat", "sub", "aud", "iss"]}
)
```

**JWT Security Checklist:**

```
[ ] Short expiry: access tokens 15 minutes, refresh tokens 7-30 days
[ ] RS256 or ES256 for multi-service; HS256 only for single-service
[ ] Validate: exp (not expired), iss (known issuer), aud (this service)
[ ] Never accept alg: none
[ ] Revocation: refresh token rotation; revocation list for critical events
[ ] Sensitive data: JWT payload is base64, not encrypted — don't put PII in it
[ ] Key rotation: support multiple public keys for zero-downtime rotation
```

```python
# Reject alg: none explicitly
ALLOWED_ALGORITHMS = ["RS256", "ES256"]  # never include "none"
payload = jwt.decode(token, public_key, algorithms=ALLOWED_ALGORITHMS)
```

---

## Input Validation

**Allowlist, not blocklist. Validate at the boundary.**

```python
# Python — Pydantic validation (validate at API boundary)
from pydantic import BaseModel, Field, field_validator, EmailStr
import re

class CreateUserRequest(BaseModel):
    email: EmailStr                          # format validated
    username: str = Field(min_length=3, max_length=30, pattern=r"^[a-zA-Z0-9_]+$")
    age: int = Field(ge=0, le=150)           # range validated
    phone: str | None = None

    @field_validator("phone")
    @classmethod
    def validate_phone(cls, v):
        if v is not None and not re.match(r"^\+[1-9]\d{7,14}$", v):
            raise ValueError("Phone must be in E.164 format")
        return v
```

```java
// Java — Bean Validation (Jakarta Validation)
public class CreateUserRequest {
    @NotBlank
    @Email
    private String email;

    @NotBlank
    @Size(min = 3, max = 30)
    @Pattern(regexp = "^[a-zA-Z0-9_]+$", message = "Username: alphanumeric and underscore only")
    private String username;

    @Min(0) @Max(150)
    private Integer age;
}
```

---

## Output Encoding

**Context-aware encoding prevents XSS:**

```python
# Python — use templating engine's auto-escaping (Jinja2)
from jinja2 import Environment, select_autoescape

env = Environment(autoescape=select_autoescape(["html", "xml"]))
# {{ user_input }} → auto-escaped in HTML context

# For JSON: use json.dumps (handles escaping) — don't build JSON by string concat
import json
response_body = json.dumps({"name": user_input})

# For shell: never build shell commands with user input (see Injection)
```

```java
// Java — OWASP Java Encoder
import org.owasp.encoder.Encode;

// HTML context
String safe = Encode.forHtml(userInput);

// JavaScript context (inside <script> tags)
String safe = Encode.forJavaScript(userInput);

// URL parameter
String safe = Encode.forUriComponent(userInput);
```

---

## CORS Configuration

```python
# FastAPI — explicit CORS allowlist
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "https://myapp.com",
        "https://admin.myapp.com",
    ],  # explicit allowlist; NEVER "*" when allow_credentials=True
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["Authorization", "Content-Type"],
    max_age=3600,
)
```

```java
// Spring — per-endpoint or global CORS
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("https://myapp.com")
            .allowedMethods("GET", "POST", "PUT", "DELETE")
            .allowCredentials(true)
            .maxAge(3600);
    }
}
```

---

## Security Headers

```nginx
# Nginx — security headers
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "DENY" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Content-Security-Policy "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; connect-src 'self' https://api.myapp.com" always;
add_header Permissions-Policy "camera=(), microphone=(), geolocation=()" always;
```

| Header | Value | Purpose |
|---|---|---|
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` | Force HTTPS for 1 year |
| `Content-Security-Policy` | explicit allowlists | Prevent XSS by restricting resource origins |
| `X-Frame-Options` | `DENY` or `SAMEORIGIN` | Prevent clickjacking |
| `X-Content-Type-Options` | `nosniff` | Prevent MIME type sniffing |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | Limit referrer info leakage |
| `Permissions-Policy` | deny unused APIs | Limit browser feature access |

---

## SAST Tools

```bash
# Python — Bandit
pip install bandit
bandit -r src/ -ll              # level: low/medium/high
bandit -r src/ -f json -o bandit-report.json

# Java — SpotBugs + Find Security Bugs
# pom.xml: add spotbugs-maven-plugin with findsecbugs-plugin
mvn spotbugs:check

# Kotlin — Detekt with security ruleset
detekt --input src/ --report html:detekt-report.html

# JavaScript/TypeScript — ESLint with security plugin
npm install eslint-plugin-security
# .eslintrc: "plugins": ["security"], "extends": ["plugin:security/recommended"]

# Rust — Clippy (built-in) + cargo-audit
cargo clippy -- -D warnings
cargo audit
```

**CI integration (GitHub Actions):**

```yaml
- name: Security Scan
  uses: github/codeql-action/analyze@v3
  with:
    languages: python, java
    queries: security-extended  # includes OWASP queries
```

---

## Container Security

```dockerfile
# GOOD: multi-stage build, non-root user, minimal base
FROM python:3.12-slim AS builder
WORKDIR /build
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# Use distroless or slim final image
FROM gcr.io/distroless/python3-debian12
COPY --from=builder /install /usr/local
COPY src/ /app

# Non-root user
USER nonroot:nonroot

WORKDIR /app
EXPOSE 8080
ENTRYPOINT ["python", "main.py"]
```

```yaml
# Kubernetes — security context
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001
        fsGroup: 10001
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: app
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
          volumeMounts:
            - name: tmp
              mountPath: /tmp       # writable volume for temp files only
      volumes:
        - name: tmp
          emptyDir: {}
```

**Container security checklist:**
```
[ ] Non-root user (UID >= 1000)
[ ] Read-only root filesystem
[ ] No privileged containers
[ ] No capabilities granted (drop ALL)
[ ] Minimal base image (distroless or alpine-based)
[ ] No secrets in Dockerfile or image layers
[ ] Scan image for CVEs before push: trivy image myapp:latest
[ ] Pin base image digest: FROM python:3.12.3@sha256:...
[ ] Multi-stage build: build tools not in production image
[ ] No SSH server in container
```

---

## Security Review Checklist (PR Gate)

```
Authentication & Authorization
[ ] New endpoint has authentication check
[ ] Authorization scoped to authenticated user's data
[ ] Admin endpoints have role check

Input & Output
[ ] User inputs validated (type, format, length, range)
[ ] Output encoding appropriate for context (HTML, JSON, shell)
[ ] No sensitive data logged

Secrets & Config
[ ] No hardcoded credentials, tokens, or keys
[ ] New env vars documented; none committed to repo

Dependencies
[ ] New dependency scanned for known CVEs
[ ] Version pinned

Cryptography
[ ] Passwords hashed with bcrypt/argon2 (not MD5/SHA1)
[ ] Sensitive data encrypted at rest where required
[ ] TLS for all external connections

Infrastructure
[ ] No new public S3 buckets / storage
[ ] Firewall rules: least privilege
[ ] New container image scanned
```
