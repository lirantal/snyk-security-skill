# Code Security (SAST) — snyk code test

Snyk Code performs static analysis (SAST) on source code to find security vulnerabilities
introduced by the developer — things like SQL injection, XSS, hardcoded secrets, insecure
crypto usage, and more. This is distinct from dependency scanning; it analyzes the code
you wrote.

## Running the Scan

```bash
# Scan the current project
snyk code test

# Scan a specific path
snyk code test ./src

# Output as JSON for programmatic use
snyk code test --json

# Save results to a file
snyk code test --json-file-output=code-results.json

# Only fail on high/critical severity
snyk code test --severity-threshold=high
```

Snyk Code supports: JavaScript/TypeScript, Python, Java, C/C++, C#, Go, PHP, Ruby, Swift,
Kotlin, Scala, and more. It auto-detects the language.

## Interpreting Results

Each finding includes:
- **Severity** (Critical/High/Medium/Low)
- **Issue type** (e.g., "SQL Injection", "Cross-site Scripting", "Hardcoded Secret")
- **File path and line number**
- **Data flow** — how untrusted input flows to the vulnerable sink
- **Fix guidance**

Prioritize findings by severity. For each High/Critical issue, trace the data flow
shown in the output to understand the attack vector before suggesting a fix.

## Common Findings and How to Fix Them

### Injection (SQL, Command, LDAP)
Use parameterized queries or prepared statements. Never concatenate user input into queries.
```javascript
// Vulnerable
db.query(`SELECT * FROM users WHERE id = ${userId}`)
// Fixed
db.query('SELECT * FROM users WHERE id = ?', [userId])
```

### Cross-Site Scripting (XSS)
Encode output based on context. Use a sanitization library (e.g., DOMPurify for HTML).
Never render unescaped user input in HTML, JS contexts, or attributes.

### Hardcoded Secrets
Move secrets to environment variables or a secrets manager. Never commit API keys,
passwords, or tokens to source control.

### Path Traversal
Validate and normalize file paths. Ensure paths resolve within the intended directory
using `path.resolve()` and checking the result starts with the expected base path.

### Insecure Deserialization
Avoid deserializing untrusted data. Use safe parsers (JSON.parse with schema validation,
YAML.safe_load). If deserialization is required, use allow-lists for permitted types.

---

## Secure Code Writing Rules

Beyond what Snyk detects, apply these security rules when writing or reviewing code.
These are organized by security domain. Reference specific sections when the user is
writing code in that area.

### Authentication & Sessions

- Set session cookies with `Secure`, `HttpOnly`, and `SameSite` attributes
- Use `__Host-` prefix with `Path=/` and no `Domain` attribute for enhanced security
- Enforce CSRF tokens on all state-changing requests
- Implement token revocation using blacklists or short-lived tokens with refresh patterns
- Generate session tokens with cryptographically secure RNG, minimum 128 bits of entropy
- Always validate JWTs server-side using the library's `verify()` method — never `decode()`
- Explicitly reject JWTs with `alg: "none"` in the header
- Validate all JWT claims: `iss`, `sub`, `aud`, `exp`, `iat`
- Prefer RS256 or ES256 over HS256; if HS256 is required, use at least 64 random chars for secret
- Validate the `iss` (issuer) parameter in OAuth responses when multiple Authorization Servers
- Implement PKCE with `code_challenge_method=S256` for public OAuth clients
- Strictly validate `redirect_uri` against a pre-registered allowlist using exact string matching

### Access Control

- Enforce authorization server-side at the service layer — never trust client-supplied tokens alone
- Use RBAC with the principle of least privilege
- Check both horizontal (user A can't access user B's data) and vertical (no privilege escalation) for every endpoint
- Restrict admin interfaces with network ACLs, VPN requirements, and MFA

### Input Validation & Output Encoding

- Validate all inputs using allow-lists for expected types, formats, and ranges
- Use schema validation libraries (AJV, Joi, JSON Schema) for structured data
- Implement context-aware output encoding: HTML, HTML attributes, JavaScript, CSS, URL each require different encoding
- Use DOMPurify for HTML sanitization
- Prevent prototype pollution: sanitize object keys, avoid unsafe deep merges with untrusted input, use `Object.create(null)` for dictionaries
- Protect against mass assignment: use DTOs with explicit field mapping, never bind request params directly to model objects
- Implement HTTP parameter pollution defenses: deduplicate parameters, validate only expected params are processed

### API Security (REST)

- Enforce `additionalProperties: false` in all OpenAPI schemas
- Set request body size limits (e.g., 1 MiB via middleware)
- Accept only declared `Content-Type`, reject others with HTTP 415
- GET/HEAD/OPTIONS/TRACE must be read-only
- Never expose secrets or PII in URL paths or query parameters — use headers
- Verify user roles and ownership on every request via authorization middleware

### Cryptography

- Use FIPS 140-2 approved algorithms for all cryptographic operations
- Use approved JWT signing algorithms: RS256, RS384, RS512, ES256, ES384, ES512
- Integrate with an enterprise secrets manager (e.g., AWS Secrets Manager, HashiCorp Vault)
- Never hardcode secrets, keys, or credentials in source code

### Frontend Security

- Implement HSTS: `Strict-Transport-Security: max-age=31536000; includeSubDomains`
- Set a restrictive CSP: `default-src 'self'`, use nonce or hash for inline scripts
- Set `X-Frame-Options: DENY` or `SAMEORIGIN` plus `frame-ancestors` in CSP
- Set `X-Content-Type-Options: nosniff`
- Set appropriate `Referrer-Policy` (e.g., `strict-origin-when-cross-origin`)
- Set `Cache-Control: no-store, no-cache, must-revalidate` for sensitive pages
- Validate all URL redirects against an allowlist of permitted domains
- Avoid storing sensitive data in `localStorage`, `sessionStorage`, or unprotected cookies
- Clear all client-side storage on logout

### Database

- Configure all DB connections with TLS 1.2+ and `sslmode=require`
- Always use parameterized queries or prepared statements — never string concatenation
- Create dedicated DB users with minimal permissions per service
- Implement row-level security where user data segregation is required

### File Handling

- Validate file types using MIME type detection on file content (not extension)
- Implement file size limits and per-user storage quotas
- Store uploads outside the web root with restricted permissions
- Prevent path traversal: validate all paths with `path.resolve()` and check they stay within intended directory
- Validate compressed file sizes before extraction; enforce limits to prevent zip bombs
- Reject archives containing symbolic links

### WebSockets

- Use WSS only (TLS 1.2+)
- Validate `Origin` header during handshake against an allowlist
- Implement per-connection rate limiting
- Require HTTP authentication before establishing WebSocket connections
- Enforce RBAC at the message level

### XML & Template Engines

- Disable external entity processing, DTD processing, and XInclude in XML parsers
- Use parameterized XPath — never dynamic XPath string concatenation with user input
- Use template engines with automatic escaping enabled (Handlebars, Jinja2 autoescape)
- Avoid `eval()` and similar dynamic code execution functions

### Sensitive Data

- Use structured logging with automatic redaction of sensitive fields
- Never log full request/response bodies containing PII or credentials
- Apply data classification — track what data is stored where
