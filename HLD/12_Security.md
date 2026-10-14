# 🔒 Security in Distributed Systems

> *"Security is not a product, but a process."* — Bruce Schneier

---

## 📋 Table of Contents
1. [Authentication vs Authorization](#auth-vs-authz)
2. [Authentication Mechanisms](#authentication-mechanisms)
3. [OAuth 2.0 Deep Dive](#oauth-20)
4. [JWT (JSON Web Tokens)](#jwt)
5. [API Security](#api-security)
6. [Zero Trust Architecture](#zero-trust)
7. [Secrets Management](#secrets-management)
8. [Encryption in Transit: TLS/SSL](#tls)
9. [Encryption at Rest](#encryption-at-rest)
10. [OWASP Top 10](#owasp)
11. [Network Security](#network-security)
12. [IAM (Identity and Access Management)](#iam)
13. [Audit Logging](#audit-logging)
14. [Real-Life Scenario](#real-life-scenario)
15. [🔥 Big Tech Story: Uber 2022 Breach](#big-tech-story)
16. [Trade-offs & Best Practices](#trade-offs)
17. [Security Checklist](#security-checklist)
18. [Key Takeaways](#key-takeaways)

---

## 🔑 Authentication vs Authorization

The most fundamental distinction in security.

**Authentication (AuthN):** Who are you? Verify identity.
```
User provides: email + password, OAuth token, certificate, biometric
System verifies: "Yes, this is Alice."
```

**Authorization (AuthZ):** What can you do? Verify permissions.
```
Alice is authenticated. Can she access /admin/delete_user?
System checks: Alice has role "user", not "admin" → 403 Forbidden
```

Common confusion: `401 Unauthorized` = not authenticated. `403 Forbidden` = authenticated but not authorized.

---

## 🔐 Authentication Mechanisms

### Passwords (And Why They're Problematic)

**Never store plaintext passwords.** Store a salted hash.

**Hashing algorithms for passwords:**

| Algorithm | Speed | Recommended | Notes |
|---|---|---|---|
| MD5 | Very fast | ❌ Never | Broken, GPU brute-force trivial |
| SHA-256 | Fast | ❌ Never | Too fast — brute-force feasible |
| bcrypt | Slow | ✅ Yes | Cost factor adjustable (default 12) |
| scrypt | Slow + memory-hard | ✅ Yes | Resistant to ASIC attacks |
| Argon2 | Slow + memory-hard | ✅ Best | 2015 Password Hashing Competition winner |

```python
import bcrypt

# Registration
def hash_password(password: str) -> bytes:
    salt = bcrypt.gensalt(rounds=12)  # 12 = 2^12 = 4096 iterations
    return bcrypt.hashpw(password.encode(), salt)

# Login verification
def verify_password(password: str, hashed: bytes) -> bool:
    return bcrypt.checkpw(password.encode(), hashed)

# Salt: bcrypt includes salt in the hash output
# Never store salt separately — it's embedded in the hash
```

**Argon2 (recommended for new systems):**
```python
from argon2 import PasswordHasher

ph = PasswordHasher(
    time_cost=2,      # 2 iterations
    memory_cost=65536,  # 64MB of memory
    parallelism=2       # 2 parallel threads
)

hash = ph.hash("user_password")
ph.verify(hash, "user_password")  # True or raises VerifyMismatchError
```

### Multi-Factor Authentication (MFA)

Something you know (password) + something you have (phone) + something you are (biometric).

**TOTP (Time-based One-Time Password):**
```
1. Server generates shared secret (base32 encoded QR code)
2. User scans with Authenticator app (Google Authenticator, Authy)
3. Every 30 seconds, app generates 6-digit code:
   TOTP = HOTP(secret, floor(current_time / 30))
4. Server validates: compute expected TOTP, compare (allow ±1 window for clock drift)
```

**WebAuthn (Passkeys):** Browser-level public key cryptography.
1. Registration: device generates key pair; public key stored on server
2. Authentication: server sends challenge; device signs with private key (biometric required)
3. No password ever transmitted; phishing-resistant (private key never leaves device)

---

## 🔄 OAuth 2.0 Deep Dive

OAuth 2.0 is an **authorization** framework (not authentication — that's OpenID Connect).

**Use case:** Allow users to grant third-party apps access to their data without sharing passwords.

### OAuth 2.0 Roles

- **Resource Owner:** The user (who owns the data)
- **Client:** The app requesting access (e.g., "Sign in with Google" button on your app)
- **Authorization Server:** Issues access tokens (e.g., Google's auth server)
- **Resource Server:** Hosts the user's data (e.g., Google's API for profile data)

### Authorization Code Flow (Most Common — for Web Apps)

```
1. User clicks "Sign in with Google" on Your App

2. Your App redirects user to Google Authorization Server:
   GET https://accounts.google.com/o/oauth2/auth
     ?client_id=your-client-id
     &redirect_uri=https://yourapp.com/callback
     &response_type=code
     &scope=openid email profile
     &state=random-csrf-token

3. User logs into Google, approves your app's permission request

4. Google redirects back to Your App with authorization code:
   GET https://yourapp.com/callback?code=AUTH_CODE&state=random-csrf-token

5. Your App exchanges code for tokens (server-to-server, never exposed to browser):
   POST https://oauth2.googleapis.com/token
     client_id=your-client-id
     client_secret=your-client-secret
     code=AUTH_CODE
     grant_type=authorization_code
     redirect_uri=https://yourapp.com/callback

6. Google returns:
   {
     "access_token": "ya29.xxx",     # Short-lived (1 hour)
     "refresh_token": "1//xxx",      # Long-lived, used to get new access tokens
     "id_token": "eyJxx.xxx.xxx",   # JWT with user info (OpenID Connect)
     "expires_in": 3600
   }

7. Your App uses access_token to call Google APIs:
   GET https://www.googleapis.com/oauth2/v1/userinfo
   Authorization: Bearer ya29.xxx
```

### PKCE (Proof Key for Code Exchange)

For mobile and single-page apps (can't safely store client_secret), PKCE prevents authorization code interception attacks.

```
1. App generates code_verifier (random 128-char string)
2. code_challenge = BASE64URL(SHA256(code_verifier))
3. Include code_challenge in authorization request
4. When exchanging code for token, include code_verifier
5. Auth server verifies: SHA256(code_verifier) == code_challenge stored earlier
```

If attacker intercepts the authorization code, they can't exchange it (don't have the code_verifier).

### Client Credentials Flow (Machine-to-Machine)

No user involved. Service A authenticates to Service B directly.

```
Service A → Auth Server: POST /token
  client_id=service-a-id
  client_secret=service-a-secret
  grant_type=client_credentials
  scope=read:orders

Auth Server → Service A: { "access_token": "...", "expires_in": 3600 }

Service A → Service B API: GET /orders
  Authorization: Bearer {access_token}
```

Used for: microservice-to-microservice communication, CI/CD pipelines accessing APIs.

---

## 🎫 JWT (JSON Web Tokens)

JWT is a compact, self-contained token format for transmitting claims between parties.

### Structure

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ1c2VyXzEyMyIsIm5hbWUiOiJBbGljZSIsInJvbGUiOiJ1c2VyIiwiaWF0IjoxNzA1MzEyMDAwLCJleHAiOjE3MDUzMTU2MDB9.signature

Three parts separated by dots:
  Header.Payload.Signature

Header (base64url encoded):
{
  "alg": "RS256",   # Signing algorithm
  "typ": "JWT"
}

Payload (base64url encoded):
{
  "sub": "user_123",           # Subject (user ID)
  "name": "Alice",
  "role": "user",
  "iat": 1705312000,           # Issued at (Unix timestamp)
  "exp": 1705315600,           # Expires at (1 hour later)
  "iss": "https://auth.myapp.com",  # Issuer
  "aud": "https://api.myapp.com"    # Audience
}

Signature:
  RS256: RSA_Sign(private_key, base64url(header) + "." + base64url(payload))
```

**JWT is NOT encrypted** (unless you use JWE). The payload is base64 encoded — readable by anyone.

**Never put sensitive data in JWT payload!** No passwords, credit cards, SSNs.

### Signing Algorithms

**HS256 (HMAC-SHA256):**
- Symmetric: same secret used to sign and verify
- All services must share the secret
- Problem: any service that can verify JWTs can also create them (forge tokens)
- Use for: single-service auth, or when all verifying services are trusted equally

**RS256 (RSA-SHA256):**
- Asymmetric: private key signs, public key verifies
- Auth server keeps private key; all other services only need public key
- Services that verify tokens CANNOT create them
- Use for: multi-service architectures (recommended for production)

**ES256 (ECDSA-SHA256):**
- Like RS256 but with Elliptic Curve crypto
- Smaller signatures (64 bytes vs 256 bytes)
- Faster verification
- Use for: high-throughput systems

### JWT Storage

| Storage | XSS Risk | CSRF Risk | Notes |
|---|---|---|---|
| localStorage | ⚠️ High | ✅ Low | JS can read it → XSS can steal token |
| sessionStorage | ⚠️ High | ✅ Low | Same as localStorage |
| httpOnly Cookie | ✅ Low | ⚠️ Medium | JS can't read → add CSRF protection |
| Memory (JS var) | ✅ Low | ✅ Low | Lost on page refresh → poor UX |

**Recommendation:** `httpOnly`, `Secure`, `SameSite=Strict` cookies with CSRF tokens.

### Refresh Tokens and Token Rotation

Access tokens should be short-lived (15 minutes). Refresh tokens are long-lived (90 days) and stored securely.

```
Access Token expired → App uses Refresh Token → Gets new Access + Refresh Token
                                                (token rotation: old refresh token invalidated)

If attacker steals refresh token:
  They get new tokens
  BUT the next time legitimate user refreshes → refresh token already used
  → Server detects rotation violation → invalidates all tokens for this user
  → User must log in again
```

### JWT Pitfalls

1. **Algorithm confusion attack:** JWT header specifies `"alg": "none"` → no signature verification. Always explicitly specify expected algorithm.

2. **Weak secret:** If using HS256, weak secrets can be brute-forced. Use cryptographically random 256-bit secrets.

3. **Missing expiry:** JWTs without `exp` claim are valid forever. Always set expiry.

4. **Not validating claims:** Check `iss`, `aud`, `exp`, `iat` on every request. A JWT for another service can be replayed.

5. **No revocation:** JWTs are stateless — you can't "invalidate" them before expiry. Solution: maintain a revocation list (blocklist) or use short-lived tokens + refresh.

---

## 🔌 API Security

### API Keys

Simple credentials for identifying and authenticating API clients.

```
curl -H "Authorization: Bearer sk_live_abc123..." https://api.example.com/data
```

**Best practices:**
- Generate cryptographically random API keys (not UUIDs — too short)
- Hash keys before storing (like passwords) — only store hash, not plaintext
- Prefix with provider name: `sk_live_` (Stripe), `ghp_` (GitHub)
- Allow key rotation and revocation
- Scope keys to minimum permissions

### HMAC Signatures (Request Signing)

Instead of sending a secret with every request, sign the request body/headers.

```python
import hmac, hashlib, time

def sign_request(method, path, body, api_secret):
    timestamp = str(int(time.time()))
    message = f"{timestamp}\n{method}\n{path}\n{body}"
    signature = hmac.new(
        api_secret.encode(),
        message.encode(),
        hashlib.sha256
    ).hexdigest()
    return timestamp, signature

# Headers sent:
# X-Timestamp: 1705312000
# X-Signature: abc123...

# Server verifies:
# Reconstruct message from request
# Compute expected signature
# Compare (in constant time to prevent timing attacks)
# Also check timestamp is within 5 minutes (prevent replay attacks)
```

Used by: AWS Signature Version 4, Stripe webhook verification.

### Mutual TLS (mTLS)

Regular TLS: client verifies server certificate (one-way).
mTLS: both client AND server present certificates (two-way).

```
Service A → Service B:
  Service A presents: client certificate (proves identity)
  Service B verifies: Is client cert signed by our trusted CA? 
  Service B presents: server certificate
  Service A verifies: Is server cert signed by our trusted CA?

Both parties verified cryptographically — no passwords, no tokens needed!
```

Used by: Service mesh (Istio enforces mTLS between all services), internal API authentication.

**Certificate management:** Use a private CA (Certificate Authority) for internal services. SPIFFE/SPIRE (Secure Production Identity Framework) manages automatic certificate rotation for microservices.

---

## 🏰 Zero Trust Architecture

**Traditional perimeter model:** "Trust but verify inside the network; don't trust outside."

```
┌─────────────────────────────────────────┐
│         Corporate Network               │
│  (Inside = trusted, anything goes)      │
│                                         │
│   Workstation ──HTTP──► Internal API   │
│   (No auth needed! Same network.)       │
│                                         │
└─────────────────────────────────────────┘
```

**Zero Trust:** "Never trust, always verify. Trust no network. Authenticate every request."

```
Every request must provide:
  1. Strong identity (mTLS certificate, JWT with short TTL)
  2. Authorization (this identity has permission for this action)
  3. Device health check (is this device compliant? Up to date?)
  4. Context check (is this request from expected location/time?)

Even internal services must authenticate to each other.
Even a request from the CEO's laptop must be verified.
```

**Zero Trust principles:**
1. Verify explicitly (always authenticate and authorize)
2. Least privilege access (minimal permissions)
3. Assume breach (act as if attackers are already inside)

**Implementation:**
- mTLS between all microservices (service mesh)
- JWT with short TTL for user-facing APIs
- VPN does not grant access — individual service auth still required
- Network micro-segmentation (each service only has network access to what it needs)

---

## 🔐 Secrets Management

### What Are Secrets?

- Database passwords
- API keys (Stripe, Twilio, etc.)
- Encryption keys
- TLS private keys
- Service account credentials

### Anti-Patterns

```python
# ❌ NEVER: Secrets in code
DATABASE_URL = "postgresql://admin:super_secret_password@db.internal/prod"
STRIPE_API_KEY = "sk_live_abc123def456"

# ❌ NEVER: Secrets in environment variables directly in source code
os.environ['STRIPE_KEY']  # If the variable is set in .env committed to git

# ❌ NEVER: Secrets in config files committed to git
# config.yaml: db_password: "my_password"
```

### HashiCorp Vault

Industry-standard secrets management.

```
Architecture:
  Applications → Vault → Secrets (dynamically generated)

Dynamic secrets:
  Instead of static DB password: "mypassword123"
  Vault generates temporary credentials:
    { "username": "v-app1-xyz123", "password": "A9x3Kj...", "ttl": "1h" }
  After 1 hour → credentials automatically revoked
  
  Benefits: Even if credentials are leaked, they expire in 1 hour
  Audit trail: Every secret access is logged

Secret engines:
  Database: Dynamic credentials for PostgreSQL, MySQL, MongoDB
  AWS: Dynamic IAM credentials
  PKI: Generate TLS certificates on demand
  KV: Store static secrets securely
```

```python
# Vault Python client
import hvac

client = hvac.Client(url='https://vault.internal', token=app_token)

# Retrieve database credentials
db_creds = client.secrets.database.generate_credentials(
    name='my-app-postgres-role'
)
host = db_creds['data']['username']
password = db_creds['data']['password']
# Use credentials; they auto-expire in 1 hour
```

### AWS Secrets Manager

Managed alternative for AWS-native stacks.

```python
import boto3, json

client = boto3.client('secretsmanager')

secret = client.get_secret_value(SecretId='prod/myapp/db-password')
creds = json.loads(secret['SecretString'])

db_password = creds['password']
```

**Auto-rotation:** AWS Secrets Manager can automatically rotate RDS database passwords. Rotation Lambda changes the DB password and updates the secret — zero downtime with a brief overlap window.

### Kubernetes Secrets (with caveats)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  password: bXlzZWNyZXQ=  # base64 encoded (NOT encrypted by default!)
```

⚠️ Kubernetes Secrets are only base64 encoded, not encrypted, by default. Enable **encryption at rest** for Secrets using a KMS provider.

Better: Use Vault's Kubernetes auth method to fetch real secrets at runtime, or use **External Secrets Operator** to sync from AWS Secrets Manager/Vault into Kubernetes Secrets.

---

## 🔒 Encryption in Transit: TLS/SSL Deep Dive

### TLS 1.3 Handshake (Simplified)

```
Client → Server: ClientHello
  - TLS version (1.3)
  - Client random
  - Cipher suites supported (e.g., TLS_AES_256_GCM_SHA384)
  - Key share (Diffie-Hellman public key)

Server → Client: ServerHello + Certificate + CertificateVerify + Finished
  - TLS version
  - Server random
  - Chosen cipher suite
  - Server's DH public key
  - Server's certificate chain (signed by trusted CA)
  - CertificateVerify: signature proving server owns the private key
  - Finished: HMAC of entire handshake so far

Client:
  - Verifies certificate chain against trusted CAs (root CAs in browser/OS)
  - Verifies CertificateVerify signature
  - Computes session keys using DH key exchange
  (Both sides now have the same session keys)

Client → Server: Finished (HMAC of handshake)
Client → Server: [Encrypted application data]
```

**TLS 1.3 improvements over 1.2:**
- 1-RTT handshake (vs 2-RTT in TLS 1.2) — faster
- 0-RTT resumption — for session resumption (replay attack risk; use with care)
- Removed weak ciphers (RC4, MD5, SHA-1, DES, 3DES, RSA key exchange)
- Forward secrecy mandatory (DHE/ECDHE only)

### Forward Secrecy (Perfect Forward Secrecy)

Even if the server's private key is stolen in the future, past sessions cannot be decrypted.

**Without PFS (RSA key exchange):**
```
Attacker records encrypted traffic today
Later gets server private key
→ Decrypts all recorded traffic retroactively
```

**With PFS (DHE/ECDHE):**
```
Each session generates ephemeral DH keys (discarded after session)
Server private key only proves server identity — not used for key derivation
Recorded traffic cannot be decrypted even with server private key
```

TLS 1.3 mandates forward secrecy.

### Certificate Management

**Let's Encrypt:** Free, automated TLS certificates.
```bash
# Certbot automatic certificate issuance and renewal
certbot --nginx -d example.com -d www.example.com
# Auto-renewal: certbot renew (add to cron)
```

**AWS ACM (Certificate Manager):**
- Free for use with AWS services (ALB, CloudFront)
- Automatic renewal
- No private key management needed

**HSTS (HTTP Strict Transport Security):**
```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```
Tells browsers: only connect via HTTPS for the next year (even if the user types http://).

Submit domain to [HSTS Preload List](https://hstspreload.org/) to be hardcoded into browsers.

---

## 🔏 Encryption at Rest

### AES-256

The standard symmetric encryption algorithm. 256-bit key → 2^256 possible keys → infeasible to brute-force.

**Used for:** Database encryption, file encryption, S3 server-side encryption.

### Envelope Encryption (AWS KMS Model)

Encrypting large data with KMS directly is expensive and slow. Envelope encryption solves this:

```
1. KMS generates a Data Encryption Key (DEK)
2. Use DEK (locally, fast) to encrypt your data (AES-256)
3. KMS encrypts the DEK itself with the Customer Master Key (CMK)
4. Store: encrypted_data + encrypted_DEK (together)

To decrypt:
1. Send encrypted_DEK to KMS
2. KMS decrypts DEK (using CMK, which never leaves KMS)
3. Use DEK to decrypt your data locally
4. Discard DEK from memory

The CMK never leaves KMS. Audit log: every decrypt operation logged.
```

**S3 Server-Side Encryption:**
```
SSE-S3: S3 manages keys (transparent, minimal control)
SSE-KMS: KMS manages keys (audit trail, cross-account, rotation control)
SSE-C: Customer provides keys (maximum control, you manage keys)
```

---

## 🛡️ OWASP Top 10 (2021)

The Open Web Application Security Project's top 10 most critical web application security risks.

### 1. Broken Access Control

Users can access resources or perform actions they shouldn't.

```python
# ❌ VULNERABLE: No authorization check
@app.route('/api/orders/<order_id>')
def get_order(order_id):
    return db.query(f"SELECT * FROM orders WHERE id = {order_id}")
    # Any authenticated user can see any order!

# ✅ SECURE: Check ownership
@app.route('/api/orders/<order_id>')
@require_auth
def get_order(order_id):
    order = db.query("SELECT * FROM orders WHERE id = ? AND user_id = ?",
                     [order_id, current_user.id])
    if not order:
        return 403  # Or 404 to not reveal existence
    return order
```

### 2. Cryptographic Failures

Using weak encryption, storing sensitive data unencrypted, transmitting over HTTP.

- Store passwords with bcrypt/Argon2, not SHA-256
- Encrypt PII in the database
- Use HTTPS everywhere; redirect HTTP to HTTPS

### 3. Injection (SQL, NoSQL, Command)

```python
# ❌ SQL Injection
query = f"SELECT * FROM users WHERE email = '{user_input}'"
# If user_input = "' OR 1=1 --" → returns all users!

# ✅ Parameterized queries
cursor.execute("SELECT * FROM users WHERE email = ?", [user_input])
```

```python
# ❌ Command injection
os.system(f"convert {uploaded_filename} output.jpg")
# If filename = "; rm -rf / ;" → destroys server!

# ✅ Use safe APIs
subprocess.run(["convert", uploaded_filename, "output.jpg"], check=True)
```

### 4. Insecure Design

Architecture-level flaws: no rate limiting, no fraud controls, insufficient threat modeling.

### 5. Security Misconfiguration

Default credentials, verbose error messages revealing internals, missing security headers.

```nginx
# Security headers in Nginx
add_header X-Frame-Options "DENY";
add_header X-Content-Type-Options "nosniff";
add_header Referrer-Policy "strict-origin-when-cross-origin";
add_header Content-Security-Policy "default-src 'self'";
add_header Permissions-Policy "geolocation=(), microphone=()";
```

### 6. Vulnerable and Outdated Components

Using libraries with known vulnerabilities.

Solution: Use dependabot/Snyk/npm audit. Pin dependency versions. Automated vulnerability scanning in CI/CD.

### 7. Identification and Authentication Failures

Weak passwords allowed, no MFA, session tokens in URLs, sessions not invalidated on logout.

### 8. Software and Data Integrity Failures

CI/CD pipeline attacks (supply chain), auto-update without signature verification.

### 9. Security Logging and Monitoring Failures

No audit logs, no alerting on suspicious activity, attackers operate undetected for months.

### 10. Server-Side Request Forgery (SSRF)

```python
# ❌ SSRF Vulnerable
def fetch_url(url):
    return requests.get(url)  # User provides URL

# User provides: http://169.254.169.254/latest/meta-data/iam/security-credentials/
# → Fetches AWS instance metadata with IAM credentials!

# ✅ Validate and allowlist URLs
def fetch_url(url):
    parsed = urlparse(url)
    if parsed.hostname not in ALLOWED_HOSTS:
        raise ValueError("Host not allowed")
    if parsed.scheme not in ['https']:
        raise ValueError("Only HTTPS allowed")
    return requests.get(url, timeout=5)
```

---

## 🌐 Network Security

### VPC (Virtual Private Cloud)

AWS VPC isolates your resources in a private network:

```
VPC: 10.0.0.0/16

Public Subnet: 10.0.1.0/24
  → Internet Gateway attached
  → Load Balancer (public-facing)
  → NAT Gateway (for private subnet egress)

Private Subnet: 10.0.2.0/24
  → No direct internet access
  → App servers (EC2)
  → Only reachable from public subnet (via LB)
  → Can reach internet via NAT Gateway (for package updates, API calls)

Data Subnet: 10.0.3.0/24
  → Most restrictive
  → Databases (RDS, ElastiCache)
  → Only reachable from private subnet
  → No internet access at all
```

### Security Groups (Stateful Firewall)

```
# App server security group:
Inbound:
  Allow port 8080 from Load Balancer security group only
  Allow port 22 (SSH) from bastion host security group only
Outbound:
  Allow port 5432 to database security group (PostgreSQL)
  Allow port 443 to 0.0.0.0/0 (HTTPS to external APIs)
  Allow port 6379 to Redis security group

# Database security group:
Inbound:
  Allow port 5432 from App server security group only
  # Nothing else!
```

Security groups are stateful: if you allow inbound on port 80, the response traffic is automatically allowed outbound (no explicit rule needed).

### Private Endpoints (VPC Endpoints)

Connect to AWS services (S3, DynamoDB) without traffic leaving AWS network.

```
Without VPC Endpoint: App → Internet Gateway → AWS S3 (traffic on public internet)
With VPC Endpoint:    App → VPC Endpoint → AWS S3 (private network, never leaves AWS)
```

Faster, cheaper (no data transfer costs), more secure (no internet exposure).

---

## 👤 IAM — Identity and Access Management

### Least Privilege Principle

Every user, service, and process should have the minimum permissions needed to do their job.

```
❌ Bad IAM policy:
{
  "Effect": "Allow",
  "Action": "*",
  "Resource": "*"
}

✅ Good IAM policy (S3 read-only for specific bucket):
{
  "Effect": "Allow",
  "Action": ["s3:GetObject", "s3:ListBucket"],
  "Resource": [
    "arn:aws:s3:::my-app-bucket",
    "arn:aws:s3:::my-app-bucket/*"
  ]
}
```

### IAM Roles (Instead of IAM Users for Services)

```
❌ Don't do this:
  Create IAM user "my-app-prod" with access keys
  Store access keys in environment variables
  Keys never rotate, can be leaked

✅ Do this:
  Assign IAM Role to EC2 instance / ECS task / Lambda
  Application uses Instance Metadata Service to get temporary credentials
  Credentials auto-rotate every 15 minutes
  No long-lived credentials to store or leak
```

---

## 📋 Audit Logging

Every security-sensitive action must be logged:

```python
# What to log:
{
  "timestamp": "2024-01-15T10:30:00Z",
  "request_id": "req_abc123",
  "user_id": "user_456",
  "ip_address": "203.0.113.5",
  "action": "user.delete",
  "resource": "user_789",
  "result": "success",
  "user_agent": "Mozilla/5.0...",
  "session_id": "sess_xyz"
}
```

**Critical events to log:**
- Authentication (success and failure)
- Authorization failures
- User data access (especially PII)
- Admin actions
- Password/credential changes
- API key creation/deletion

**What NOT to log:**
- Passwords (even failed attempts)
- Full credit card numbers
- Sensitive PII in detail

**Tools:** AWS CloudTrail (AWS API audit), AWS CloudWatch Logs, ELK stack, Datadog.

---

## 💼 Real-Life Scenario: Securing a Fintech API

**Requirements:** Payment processing API serving millions of users. PCI-DSS compliance required.

**Architecture decisions:**

```
External Layer:
  Cloudflare → WAF (block OWASP Top 10), DDoS protection, Bot management
  
API Gateway:
  Rate limiting: 100 req/min per API key; 10 req/min on payment endpoints
  JWT validation: verify signature, check exp, iss, aud
  IP allowlisting for B2B partners
  
Authentication:
  OAuth 2.0 + OIDC for user-facing clients (Authorization Code + PKCE)
  Client credentials + mTLS for B2B partners
  JWT access tokens: 15-minute TTL, RS256
  Refresh tokens: 90-day TTL, rotation + reuse detection
  MFA required for all admin users (TOTP or WebAuthn)
  
App Services:
  mTLS between all microservices
  Vault for dynamic database credentials (hourly rotation)
  No environment variable secrets (Vault agent sidecar)
  
Database (PCI scope):
  AES-256 encryption at rest (RDS encrypted with KMS CMK)
  Card data encrypted at field level (not just disk encryption)
  PostgreSQL: row-level security (users can only see their own records)
  Network: separate VPC subnet, security group allows only app servers
  
Audit:
  Every payment operation logged with user_id, amount, card_bin, ip, result
  AWS CloudTrail for all IAM/AWS API calls
  SIEM (Splunk) alerts on: login failures > 5/minute, admin access at unusual hours
  
Compliance:
  Regular penetration testing (quarterly)
  Automated vulnerability scanning in CI/CD (Snyk)
  Dependency pinning + security updates automated (Dependabot)
```

---

## 🔥 Big Tech Story: The Uber 2022 Breach

### What Happened

In September 2022, a hacker (reportedly an 18-year-old) gained access to Uber's internal systems, including Slack, AWS, HackerOne, Google Workspace admin, and more.

**The attack vector:** Social engineering — not sophisticated hacking.

### Step-by-Step Attack

1. **Credential theft:** The attacker purchased Uber employee credentials from the dark web (likely from a previous breach on another site where the employee reused a password).

2. **MFA fatigue attack:** The employee's account had MFA enabled. The attacker continuously sent MFA push notifications to the employee's phone for over an hour.

3. **Social engineering:** The attacker impersonated Uber IT support on WhatsApp:
   > "Hi, I'm from Uber IT. You're getting these MFA requests because there's a security alert on your account. Please approve one to confirm your identity."
   
   The exhausted employee approved the MFA request.

4. **Initial access:** The attacker now had valid credentials + MFA approval → logged into Uber's VPN.

5. **Privilege escalation:** Once on the internal network, the attacker found a network share containing PowerShell scripts. The scripts had **hardcoded credentials** for a Privileged Access Management (PAM) system with admin access.

6. **Full compromise:** With PAM admin access, the attacker could generate credentials for any internal system. They accessed:
   - Uber's Slack workspace (posted offensive messages)
   - AWS console (access to all AWS resources)
   - Google Workspace admin
   - HackerOne bug bounty reports (sensitive vulnerability details)
   - Uber's EDR (security tool)

### Why This Was So Devastating

The combination of factors that led to complete compromise:
1. Employee credential reuse across sites
2. MFA push notification fatigue (approval out of exhaustion/confusion)
3. **Hardcoded credentials in scripts** (the most critical failure)
4. Insufficient network segmentation (one internal access → all internal access)
5. Insufficient monitoring (attack took significant time; no real-time alert)

### What We Can Learn

1. **Never hardcode credentials.** Use a secrets manager. Scripts should fetch credentials dynamically, never have them embedded.

2. **MFA push notifications are vulnerable.** Use number matching (the push shows a code that the attacker must match on the device) or WebAuthn/FIDO2 (phishing-resistant, based on cryptographic proof rather than a push).

3. **Zero Trust is not optional.** VPN access ≠ trusted. Every internal service must require authentication. The attacker's lateral movement would have been stopped by proper service-level auth.

4. **Segment your network.** An attacker in the VPN should NOT automatically have access to AWS console, HackerOne, Slack admin. Each requires separate auth.

5. **Behavioral monitoring.** A security alert should have triggered when: same user accessed 10+ internal systems in 1 hour that they've never accessed before.

6. **Secrets in code.** Run secret scanning tools (truffleHog, git-secrets, GitHub Secret Scanning) on every commit. The PAM credentials in the PowerShell script should have been caught immediately.

**The 2022 Uber breach is a masterclass in how social engineering + poor secrets management + insufficient zero trust = complete organizational compromise.** The technical controls failed not through sophisticated attacks but through basic operational security failures.

---

## ⚖️ Trade-offs & Best Practices

### Security Checklist for Production Systems

**Authentication:**
- [ ] Passwords hashed with Argon2/bcrypt (never MD5, SHA-256)
- [ ] MFA required for admin users; recommended for all users
- [ ] JWT: short TTL (15 min), RS256 signing, all claims validated
- [ ] Refresh token rotation with reuse detection
- [ ] Account lockout after failed attempts (with exponential backoff)

**Authorization:**
- [ ] Every endpoint has explicit authorization check
- [ ] Least privilege IAM roles
- [ ] No hardcoded permissions (role-based)
- [ ] Test authorization as different user roles

**Secrets:**
- [ ] No secrets in code, config files, or environment in source control
- [ ] Vault or AWS Secrets Manager for all secrets
- [ ] Secrets rotation automated
- [ ] Secret scanning in CI/CD pipeline

**Network:**
- [ ] All traffic HTTPS (redirect HTTP)
- [ ] HSTS headers set
- [ ] VPC with private subnets for databases/services
- [ ] Security groups: least privilege
- [ ] WAF in front of public endpoints

**Data:**
- [ ] Encryption at rest (AES-256, KMS)
- [ ] PII encrypted at field level for sensitive data
- [ ] Regular encrypted backups tested

**Operations:**
- [ ] Audit logging for all security events
- [ ] Alerting on suspicious patterns
- [ ] Dependency vulnerability scanning (Snyk, Dependabot)
- [ ] Regular penetration testing
- [ ] Incident response plan

---

## 🎯 Key Takeaways

| Concept | Core Insight |
|---|---|
| **AuthN vs AuthZ** | Authentication = identity; Authorization = permissions; always do both |
| **Argon2/bcrypt** | Never SHA/MD5 for passwords; use slow memory-hard algorithms |
| **OAuth 2.0** | Delegation framework; Authorization Code + PKCE for user-facing apps |
| **JWT** | Self-contained, RS256-signed, short-lived; never put secrets in payload |
| **mTLS** | Both parties present certificates; ideal for service-to-service auth |
| **Zero Trust** | Never trust the network; always verify every request |
| **Secrets Management** | Vault/AWS Secrets Manager; dynamic secrets; never hardcode |
| **TLS 1.3** | 1-RTT handshake; forward secrecy mandatory; perfect for all services |
| **Least Privilege** | Every identity gets minimum permissions; IAM roles over users for services |
| **Uber Lesson** | Social engineering + hardcoded credentials + no zero trust = total breach |

---

*Next: [13_Observability.md](13_Observability.md) — Metrics, logs, traces, and chaos engineering for running distributed systems in production.*
