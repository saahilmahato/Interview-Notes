# 🔐 Authentication & Authorization — Complete Cheatsheet

---

## 📌 Core Concepts First

### Authentication vs Authorization
- **Authentication** = *Who are you?* (verifying identity)
- **Authorization** = *What can you do?* (verifying permissions)

Think of it like a hotel:
- Authentication = showing your ID at check-in to get a key card
- Authorization = that key card only opens *your* room, not all rooms

---

## 🔹 JWT vs Sessions

### Sessions (Stateful)
```
User logs in → Server creates session → Stores it in DB/memory → 
Sends back session ID via cookie → Client sends cookie on every request → 
Server looks up session ID in DB to validate
```

**How it works:**
- Server keeps track of all logged-in users in a store (Redis, DB, memory)
- Client only holds a meaningless session ID
- Every request = a DB lookup

**Pros:**
- Easy to invalidate (just delete session from store)
- Server has full control
- Smaller data sent on each request

**Cons:**
- Stateful — hard to scale horizontally (need shared session store)
- Memory/DB pressure with many users
- Doesn't work well for microservices

---

### JWT — JSON Web Token (Stateless)
```
User logs in → Server creates signed JWT → Sends token to client → 
Client stores it → Sends JWT in Authorization header on every request → 
Server validates signature (no DB lookup needed)
```

**Structure — 3 parts separated by dots:**
```
header.payload.signature

eyJhbGciOiJIUzI1NiJ9   ← Header (algorithm + type)
.eyJ1c2VySWQiOiIxMjMifQ  ← Payload (claims/data)
.SflKxwRJSMeKKF2QT4fwpM  ← Signature (server verifies this)
```

**Header:**
```json
{ "alg": "HS256", "typ": "JWT" }
```

**Payload (Claims):**
```json
{
  "sub": "userId_123",
  "email": "user@example.com",
  "role": "admin",
  "iat": 1700000000,   ← issued at
  "exp": 1700003600    ← expires at (1 hour later)
}
```

**Signature:**
```
HMACSHA256(base64(header) + "." + base64(payload), SECRET_KEY)
```
The server uses its secret key to verify nobody tampered with the token.

**⚠️ Important:** JWT payload is **base64 encoded, NOT encrypted**. Anyone can decode and read it. Never put passwords or sensitive data in JWT payload.

**Pros:**
- Stateless — no DB lookup needed on each request
- Works great for microservices (any service can verify token with shared secret)
- Scalable

**Cons:**
- **Cannot be easily invalidated** before expiry (this is the big one)
- Larger payload sent on every request
- If secret key is leaked, all tokens are compromised

---

### JWT vs Sessions — Side by Side

| | Sessions | JWT |
|---|---|---|
| State | Stateful | Stateless |
| Storage (server) | DB / Redis | Nothing |
| Storage (client) | Cookie | localStorage / Cookie |
| Scalability | Harder | Easier |
| Revocation | Easy (delete from DB) | Hard (need blocklist) |
| Performance | DB lookup per request | CPU for signature verification |
| Best for | Monoliths, web apps | APIs, microservices, mobile |

---

### Why Do Refresh Tokens Exist?

**The Problem:**
- Short-lived access tokens expire quickly (15 min) → good for security
- But you don't want users to log in every 15 minutes → bad UX

**The Solution — Two Token System:**

```
Access Token  → Short-lived (15 min), used to access resources
Refresh Token → Long-lived (7-30 days), used ONLY to get new access tokens
```

**Flow:**
```
1. User logs in → gets Access Token (15min) + Refresh Token (7 days)
2. User makes API requests with Access Token
3. Access Token expires → client silently sends Refresh Token to /auth/refresh
4. Server validates Refresh Token → issues new Access Token
5. If Refresh Token is compromised/expired → user must log in again
```

**Why this is secure:**
- Access Token has minimal lifetime → even if intercepted, expires soon
- Refresh Token stored securely (httpOnly cookie, not localStorage)
- Refresh Token can be rotated (new one issued each time it's used)
- Refresh Token can be revoked server-side (stored in DB)

**Refresh Token Rotation:**
Every time a refresh token is used, invalidate it and issue a new one. If an old refresh token is used, it means token theft → invalidate ALL tokens for that user.

---

## 🔹 OAuth 2.0 Basics

**What is it?** A protocol that lets users grant third-party apps access to their data *without sharing their password*.

**Real example:** "Login with Google" — you never give your Google password to the app. Google vouches for you.

### Key Roles:
- **Resource Owner** = the user
- **Client** = the application wanting access (your app)
- **Authorization Server** = issues tokens (Google, GitHub, Auth0)
- **Resource Server** = holds the protected data (Google APIs, GitHub APIs)

### Most Common Flow — Authorization Code Flow:
```
1. User clicks "Login with Google"
2. App redirects user to Google's auth page with:
   - client_id, redirect_uri, scope, state (CSRF protection)

3. User logs in on Google and approves permissions

4. Google redirects back to your app with a short-lived AUTH CODE

5. Your backend exchanges auth code for tokens:
   POST /token with (code, client_id, client_secret)

6. Google returns Access Token + Refresh Token

7. Your app uses Access Token to call Google APIs on user's behalf
```

**Why not send the token directly in step 4?**
The auth code goes through the browser (less secure). The final token exchange happens server-to-server with the client secret, which never touches the browser.

### OAuth Scopes:
Scopes limit what the app can access:
```
scope=email profile        → read email and profile
scope=repo                 → access GitHub repos
scope=read:user write:repo → granular GitHub permissions
```

Always request **minimum necessary scopes** (principle of least privilege).

### OAuth vs OpenID Connect (OIDC):
- **OAuth 2.0** = Authorization framework (access to resources)
- **OIDC** = Authentication layer ON TOP of OAuth (identity, who the user is)
- OIDC adds an **ID Token** (a JWT with user info like name, email, picture)
- "Login with Google" uses OIDC, not just OAuth

---

## 🔹 How to Secure a Payment API

This is a layered defense approach — there's no single silver bullet.

### 1. Authentication & Authorization
```
- Use short-lived JWT access tokens (15 min max)
- Require re-authentication for sensitive actions (step-up auth)
- Implement RBAC (Role-Based Access Control) — not every user can refund
- API keys for server-to-server calls, scoped to minimum permissions
```

### 2. Transport Security
```
- TLS 1.2+ everywhere, no HTTP
- Certificate pinning on mobile apps
- HSTS headers (force HTTPS)
```

### 3. Input Validation & Injection Prevention
```
- Validate and sanitize all inputs
- Use parameterized queries (prevent SQL injection)
- Validate amounts server-side — never trust client-sent price
- Idempotency keys — prevent duplicate charge on retry
```

### 4. Rate Limiting & Throttling
```
- Limit requests per user/IP/API key
- Stricter limits on payment endpoints vs read endpoints
- Implement exponential backoff for failed attempts
- Block/flag after N failed attempts
```

### 5. Fraud Detection
```
- Velocity checks — same card used 50 times in 1 min? Flag it
- Geolocation checks — login from US, payment from Russia 2 min later?
- Device fingerprinting
- Amount anomaly detection (user normally pays $10, now $10,000?)
- 3D Secure (3DS) for card payments — extra verification layer
```

### 6. Secrets & Keys Management
```
- Never hardcode API keys/secrets — use environment variables
- Rotate keys regularly
- Use a secrets manager (AWS Secrets Manager, HashiCorp Vault)
- Separate keys for prod/staging
```

### 7. Logging & Monitoring
```
- Log every payment attempt with timestamp, IP, user, amount
- Alert on anomalies in real time
- Never log card numbers, CVV, or full tokens (PCI-DSS compliance)
- Use correlation IDs to trace requests across services
```

### 8. PCI-DSS Compliance
```
- Never store raw card data — use tokenization (Stripe, Braintree handle this)
- Encrypt cardholder data at rest and in transit
- Regular security audits and penetration testing
```

### 9. Webhook Security (if receiving payment events)
```
- Verify webhook signatures (Stripe sends HMAC signature in header)
- Use HTTPS endpoints only
- Respond quickly (200 OK), process async
- Be idempotent — handle duplicate webhook delivery
```

---

## 🔹 Additional Critical Topics

### RBAC vs ABAC

**RBAC — Role-Based Access Control:**
```
User → has Role → Role → has Permissions

user.role = "admin"   → can do everything
user.role = "viewer"  → can only read
user.role = "editor"  → can read and write
```

**ABAC — Attribute-Based Access Control:**
```
Access based on attributes of user, resource, and environment

"User can edit document IF user.department == document.department 
 AND current time is business hours AND user.clearance >= document.sensitivity"
```
ABAC is more flexible but complex. Use RBAC for most apps, ABAC for complex enterprise scenarios.

---

### Password Security

```
NEVER store plain text passwords. NEVER store MD5/SHA1 hashed passwords.

✅ Use: bcrypt, scrypt, or Argon2 (these are slow by design — makes brute force hard)

bcrypt adds a salt automatically (prevents rainbow table attacks)
Work factor = how slow it is. Increase it as hardware gets faster.
```

```javascript
// bcrypt example
const hash = await bcrypt.hash(password, 12); // 12 = work factor
const isValid = await bcrypt.compare(inputPassword, storedHash);
```

---

### Multi-Factor Authentication (MFA)

Three factors:
- **Something you know** → password, PIN
- **Something you have** → phone (TOTP app), hardware key (YubiKey)
- **Something you are** → fingerprint, face

**TOTP (Time-based One-Time Password):**
```
1. Server generates secret, encodes as QR code
2. User scans with Google Authenticator / Authy
3. App generates 6-digit code every 30 seconds using: HMAC(secret + current_time_window)
4. Server generates same code independently and compares
```

---

### CSRF — Cross-Site Request Forgery

**Attack:** Malicious site tricks browser into making authenticated request to your API using stored cookies.

**Prevention:**
- CSRF tokens (unique token per session, sent in form, verified server-side)
- `SameSite=Strict` or `SameSite=Lax` cookie attribute
- Check `Origin`/`Referer` header
- For APIs using JWT in Authorization header → naturally CSRF-safe (browser doesn't auto-send headers cross-origin)

---

### CORS — Cross-Origin Resource Sharing

```
Frontend at https://myapp.com wants to call API at https://api.myapp.com

Browser blocks this by default (same-origin policy)
Server must explicitly allow it via CORS headers:

Access-Control-Allow-Origin: https://myapp.com
Access-Control-Allow-Methods: GET, POST, PUT
Access-Control-Allow-Headers: Authorization, Content-Type
```

**Never set** `Access-Control-Allow-Origin: *` on APIs that use cookies or handle sensitive data.

---

### Cookie Security Attributes

```
Set-Cookie: token=abc123;
  HttpOnly;     ← JS cannot read it (prevents XSS token theft)
  Secure;       ← only sent over HTTPS
  SameSite=Strict; ← not sent on cross-site requests (CSRF protection)
  Path=/;
  Max-Age=86400 ← expires in 1 day
```

---

### Token Storage — Where to Store JWTs?

| Location | XSS Risk | CSRF Risk | Notes |
|---|---|---|---|
| localStorage | ❌ High | ✅ Safe | JS can read it → XSS steals it |
| sessionStorage | ❌ High | ✅ Safe | Same as localStorage |
| httpOnly Cookie | ✅ Safe | ❌ Risk | Can't be read by JS, use SameSite to mitigate CSRF |

**Best practice:** Store access token in memory (JS variable), store refresh token in httpOnly cookie.

---

## 🎯 Common Interview Questions & Answers

---

**Q: What's the difference between JWT and sessions? When would you use each?**

> Sessions are stateful — the server stores session data and the client holds just an ID. JWTs are stateless — all data lives in the token and the server just verifies the signature. I'd use sessions for traditional web apps where I need immediate revocation (like banking — user changes password, all sessions die instantly). I'd use JWTs for APIs and microservices where statelessness and scalability matter — each service can independently verify a token without hitting a central session store.

---

**Q: What is the biggest problem with JWTs and how do you solve it?**

> JWTs can't be invalidated before they expire. If a user's token is stolen or they log out, the token remains valid until expiry. Solutions: (1) Keep access tokens very short-lived (15 min), (2) Use a token blocklist/denylist in Redis for critical revocations — check it on each request, (3) Refresh token rotation — if a reused refresh token is detected, revoke all sessions for that user immediately. It's a trade-off between true statelessness and security.

---

**Q: Why do refresh tokens exist?**

> To balance security and user experience. Access tokens should be short-lived to minimize damage if intercepted, but you can't ask users to log in every 15 minutes. Refresh tokens are long-lived but only used to silently get new access tokens — they're stored more securely (httpOnly cookie) and never sent to resource servers. If an access token leaks, the attacker has 15 minutes. If a refresh token leaks, you can revoke it server-side since they're typically stored in a DB.

---

**Q: Explain OAuth 2.0. Why do we use it?**

> OAuth 2.0 lets users authorize a third-party app to access their data without sharing credentials. For example, "Login with Google" — the user authenticates with Google, Google issues an auth code to my app, my backend exchanges that code for tokens server-to-server. My app never sees the user's Google password. It's delegation of access, not transfer of credentials. I use it to avoid the responsibility of storing passwords for third-party integrations and to let users control what data they share via scopes.

---

**Q: How do you secure a REST API?**

> Layered approach: (1) TLS everywhere, (2) authentication via JWT or API keys, (3) authorization via RBAC/ABAC on every endpoint, (4) input validation and sanitization, (5) rate limiting, (6) never trust client data — validate server-side, (7) proper error messages that don't leak system info, (8) logging and monitoring with anomaly alerting, (9) CORS configured strictly, (10) security headers (HSTS, CSP, X-Frame-Options).

---

**Q: How would you prevent someone from replaying a stolen JWT?**

> Several approaches: (1) Short expiration — stolen token expires soon anyway, (2) Include the user's IP or device fingerprint in the token and validate it (though this breaks on mobile networks), (3) Use the `jti` claim (JWT ID) — a unique identifier per token — stored in Redis. On use, mark it as used. Reuse = revoke all tokens. (4) Refresh token rotation with reuse detection — if old refresh token is presented, something is wrong, kill all sessions.

---

**Q: What is the difference between authentication and authorization?**

> Authentication is proving you are who you claim to be — logging in with credentials, verifying your identity. Authorization is determining what you're allowed to do once authenticated — your permissions, roles, and access levels. You can be authenticated but not authorized (logged in but trying to access admin panel). A system can have authorization without authentication in some designs (public resources with rate limits). Always do authentication before authorization.

---

**Q: What is PKCE and why does it matter?**

> PKCE (Proof Key for Code Exchange) is an extension to OAuth's Authorization Code Flow for public clients (mobile apps, SPAs) that can't securely store a client secret. Without PKCE, if an attacker intercepts the auth code, they can exchange it for tokens. With PKCE: the client generates a random `code_verifier`, hashes it to a `code_challenge`, sends the challenge with the auth request. When exchanging the code, they send the original `code_verifier`. The server hashes it and compares — only the original client can complete the exchange. Essentially replaces the client secret with a one-time cryptographic proof.

---

**Q: How does bcrypt work and why not SHA256 for passwords?**

> SHA256 is fast — that's a problem for passwords. A GPU can try billions of SHA256 hashes per second, making brute force trivial. bcrypt is intentionally slow (configurable work factor) — it takes ~100ms per hash. It also automatically generates and embeds a unique salt, preventing rainbow table attacks. As hardware improves, you increase the work factor. Argon2 is even better for new systems — it's memory-hard, making GPU/ASIC attacks even more expensive.

---

**Q: What is the difference between symmetric and asymmetric JWT signing?**

> **HS256 (symmetric)** — uses one shared secret to both sign and verify. Simple, fast, but every service that verifies tokens must have the secret. If any service is compromised, the secret leaks. **RS256 (asymmetric)** — uses a private key to sign (only auth server has it) and a public key to verify (distributed to all services). Microservices can verify tokens without ever having signing capability. This is far more secure in distributed systems. Auth server publishes public keys at a `/.well-known/jwks.json` endpoint.

---

**Q: How would you handle a situation where a user's account is compromised mid-session?**

> (1) Force logout by invalidating all refresh tokens in DB — access tokens will expire naturally within 15 min, (2) For immediate effect: maintain a token blocklist in Redis keyed by `jti`, add all active tokens when compromise is detected — overhead is only per-request Redis lookup, (3) Require re-authentication for the account, (4) Notify the user, (5) Log all recent activity for forensics. The right approach depends on your token lifetime and acceptable risk window.

---

**Q: What is the Principle of Least Privilege and how do you apply it to APIs?**

> Users and services should have access only to what they absolutely need for their function — nothing more. In practice: (1) Fine-grained RBAC — viewers can't write, writers can't delete, (2) OAuth scopes — request only the permissions you need, (3) API keys scoped to specific endpoints, (4) Internal service accounts with only the DB tables/operations they need, (5) Time-limited access — elevated permissions expire, (6) Audit access regularly and revoke what's unused.

---

**Q: What is token binding and certificate pinning?**

> **Token binding** — cryptographically binds a token to the TLS session so even if the token is stolen, it can't be replayed from a different connection. **Certificate pinning** — mobile/client apps embed expected server certificate (or its public key hash). If the certificate changes unexpectedly (e.g., MITM attack with fake cert), the app rejects the connection. Important for high-security apps like banking, though it requires careful management during certificate rotation.

---

## 🚀 Quick Reference — Security Checklist

```
Authentication:
☐ Passwords hashed with bcrypt/Argon2 (never MD5/SHA1)
☐ MFA available (especially for admin accounts)
☐ Account lockout after failed attempts
☐ Secure password reset (time-limited, single-use tokens)
☐ Short-lived access tokens + refresh tokens

Authorization:
☐ RBAC enforced on every endpoint (server-side, not just frontend)
☐ User can only access their own resources (check ownership)
☐ Admin endpoints extra protected

Transport:
☐ TLS everywhere
☐ HSTS header set
☐ Secure + HttpOnly + SameSite cookies

API:
☐ Rate limiting on all endpoints (stricter on auth/payment)
☐ Input validation + sanitization
☐ Never trust client-provided IDs/prices
☐ CORS strictly configured
☐ Idempotency keys on payment endpoints

Ops:
☐ Secrets in env vars / secrets manager (not in code)
☐ Rotate keys regularly
☐ Log all auth events (never log passwords/cards)
☐ Alerts on anomalies
☐ Regular penetration testing
```

---

This covers the full landscape. The most important mental model: **defense in depth** — assume every layer can be breached, and have the next layer ready to catch it.