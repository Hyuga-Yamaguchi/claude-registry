---
name: security
description: Language and framework-agnostic security best practices. Use this skill when implementing authentication, handling user input, working with secrets, creating APIs, or dealing with sensitive data. Based on OWASP Top 10 and industry standards.
---

# Security Best Practices

This skill provides comprehensive security guidelines applicable across all programming languages and frameworks.

## When to Use This Skill

- Implementing authentication or authorization
- Handling user input or file uploads
- Creating API endpoints
- Working with secrets, credentials, or sensitive data
- Implementing payment or financial features
- Integrating third-party services
- Building multi-tenant systems
- Deploying to production

## Security Principles (Defense in Depth)

1. **Least Privilege**: Grant minimum permissions necessary
2. **Fail Securely**: Default to deny, not allow
3. **Validate Input, Escape Output**: Trust nothing from users
4. **Defense in Depth**: Multiple layers of security
5. **Separation of Concerns**: Isolate security-critical code
6. **Security by Design**: Build security in from the start, not as an afterthought
7. **Assume Breach**: Design systems to limit damage when compromised

---

## OWASP Top 10 Security Risks

### 1. Broken Access Control

**Risk**: Users can act outside their permissions (access other users' data, escalate privileges).

**Prevention Principles**:
- Deny by default (require explicit permission grants)
- Enforce authorization checks on server side (never trust client)
- Verify user owns the resource before allowing access
- Implement Role-Based Access Control (RBAC) or Attribute-Based Access Control (ABAC)
- Log access control failures for monitoring

**Common Vulnerabilities**:
```
❌ Direct Object Reference
GET /api/users/123/profile  # Can user access user 123's profile?

❌ Path Traversal
GET /api/files?path=../../etc/passwd

❌ Missing Authorization Check
DELETE /api/users/456  # Anyone can delete any user?
```

**Secure Pattern**:
```
✅ Always verify ownership/permissions server-side
1. Extract authenticated user ID from session/token
2. Verify user has permission to access resource
3. Only then proceed with operation
4. Log authorization failures
```

**Checklist**:
- [ ] Authorization checks on ALL sensitive operations
- [ ] User can only access their own resources (or explicitly shared)
- [ ] Admin actions require admin role verification
- [ ] No reliance on client-side permission checks
- [ ] Proper session management (timeout, regeneration)

---

### 2. Cryptographic Failures

**Risk**: Sensitive data exposed due to weak/missing encryption.

**Prevention Principles**:
- Encrypt sensitive data at rest and in transit
- Use strong, proven algorithms (avoid homebrew crypto)
- Never store passwords in plaintext
- Use secure random number generators
- Implement proper key management

**Sensitive Data**:
- Passwords, tokens, API keys
- Personal Identifiable Information (PII): SSN, passport, address
- Financial data: credit cards, bank accounts
- Health records (HIPAA compliance)
- Session identifiers

**Secure Storage Pattern**:
```
✅ Passwords
- Hash with bcrypt/argon2/scrypt (adaptive algorithms)
- Use minimum work factor (bcrypt 12+, argon2id recommended)
- Never use MD5, SHA1, or plain SHA256 for passwords

✅ Secrets (API keys, tokens)
- Store in environment variables (never in source code)
- Use secret management services (AWS Secrets Manager, HashiCorp Vault)
- Rotate secrets regularly

✅ PII / Sensitive Data
- Encrypt at rest (AES-256-GCM or ChaCha20-Poly1305)
- Encrypt in transit (TLS 1.2+)
- Consider tokenization for credit cards
```

**Checklist**:
- [ ] HTTPS/TLS enforced (no plain HTTP)
- [ ] Passwords hashed with adaptive algorithm
- [ ] No secrets in source code, git history, or logs
- [ ] Encryption keys stored separately from data
- [ ] Strong random number generation for tokens
- [ ] Sensitive data encrypted at rest

---

### 3. Injection

**Risk**: Attacker injects malicious code/commands through untrusted input.

**Types**:
- SQL Injection
- Command Injection (OS commands)
- LDAP Injection
- XPath Injection
- NoSQL Injection
- Template Injection

**Prevention Principles**:
- **Never concatenate user input into commands/queries**
- Use parameterized queries / prepared statements
- Use ORMs/query builders correctly
- Validate and sanitize all input
- Apply least privilege to database accounts

**SQL Injection Example**:
```sql
❌ DANGEROUS - String concatenation
query = "SELECT * FROM users WHERE email = '" + userEmail + "'"
-- Attacker sends: ' OR '1'='1
-- Result: SELECT * FROM users WHERE email = '' OR '1'='1'

✅ SAFE - Parameterized query
query = "SELECT * FROM users WHERE email = ?"
execute(query, [userEmail])
```

**Command Injection Example**:
```bash
❌ DANGEROUS - Shell command with user input
command = "ping " + userProvidedHost
exec(command)
-- Attacker sends: google.com && rm -rf /

✅ SAFE - Use APIs, not shell commands
ping_result = ping_api(userProvidedHost)  # Use library function
```

**Checklist**:
- [ ] All queries use parameterized statements
- [ ] No string concatenation in SQL/NoSQL queries
- [ ] Avoid shell commands with user input
- [ ] Input validation with whitelists (not blacklists)
- [ ] Database user has minimal privileges (not root/admin)
- [ ] ORM usage reviewed for raw query safety

---

### 4. Insecure Design

**Risk**: Fundamental design flaws that can't be fixed with implementation changes.

**Prevention Principles**:
- Threat modeling during design phase
- Use secure design patterns
- Limit resource consumption (prevent DoS)
- Implement business logic controls
- Plan for failure scenarios

**Common Design Flaws**:
```
❌ Missing Resource Limits
- No pagination (return entire database)
- No file size limits (upload huge files)
- No rate limiting (brute force attacks)
- No timeout limits (resource exhaustion)

❌ Missing Business Logic Controls
- Unlimited password reset requests
- No duplicate transaction prevention
- Race conditions in critical operations
- No idempotency for payment operations
```

**Secure Design Pattern**:
```
✅ Implement Limits Everywhere
1. Rate limiting on all APIs (per user, per IP)
2. Pagination on all list endpoints (max 100 items)
3. File upload limits (size, type, count)
4. Request timeouts (prevent long-running attacks)
5. Circuit breakers for external dependencies

✅ Business Logic Controls
1. Idempotency keys for critical operations
2. Transaction state machines (prevent invalid state transitions)
3. Optimistic locking for concurrent updates
4. Audit logs for sensitive operations
```

**Checklist**:
- [ ] Threat model created for critical features
- [ ] Rate limiting on all endpoints
- [ ] Resource limits defined and enforced
- [ ] Business logic tested for edge cases
- [ ] Failure modes considered and handled

---

### 5. Security Misconfiguration

**Risk**: Insecure default configurations, incomplete setups, exposed admin interfaces.

**Prevention Principles**:
- Remove unnecessary features/services
- Change all default credentials
- Keep software updated
- Apply security headers
- Disable directory listing and verbose errors in production

**Common Misconfigurations**:
```
❌ Default Credentials
- admin/admin, root/root still active
- Default database passwords unchanged
- Default API keys in production

❌ Unnecessary Services
- Admin panels exposed to internet
- Development features in production
- Verbose error messages showing stack traces
- Directory listing enabled

❌ Missing Security Headers
- No Content-Security-Policy
- No X-Frame-Options (clickjacking)
- No X-Content-Type-Options
- CORS misconfigured (allows *)
```

**Security Headers (Production)**:
```
✅ Required Headers
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
Content-Security-Policy: default-src 'self'
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: geolocation=(), microphone=()
```

**Checklist**:
- [ ] All default credentials changed
- [ ] Unused features/services disabled
- [ ] Security headers configured
- [ ] Verbose error messages disabled in production
- [ ] Admin interfaces behind authentication
- [ ] Regular security updates applied
- [ ] CORS properly configured (not *)

---

### 6. Vulnerable and Outdated Components

**Risk**: Using libraries/frameworks with known vulnerabilities.

**Prevention Principles**:
- Keep dependencies up to date
- Monitor security advisories
- Remove unused dependencies
- Use Software Composition Analysis (SCA) tools
- Pin versions, use lock files

**Dependency Security**:
```
✅ Regular Updates
1. Check for security advisories weekly
2. Update dependencies monthly (at minimum)
3. Subscribe to security mailing lists
4. Use automated tools (Dependabot, Snyk, etc.)

✅ Version Pinning
1. Use lock files (package-lock.json, Gemfile.lock, etc.)
2. Pin exact versions in production
3. Test updates in staging first

✅ Minimize Dependencies
1. Remove unused packages
2. Prefer standard library over external deps
3. Audit transitive dependencies
```

**Checklist**:
- [ ] Dependency scanning enabled (npm audit, Snyk, etc.)
- [ ] No known critical vulnerabilities
- [ ] Lock files committed to version control
- [ ] Regular update schedule defined
- [ ] Automated security alerts enabled
- [ ] Production uses specific versions (not "latest")

---

### 7. Identification and Authentication Failures

**Risk**: Broken authentication allows attackers to compromise accounts.

**Prevention Principles**:
- Never roll your own authentication
- Implement multi-factor authentication (MFA)
- Use strong password policies
- Protect against brute force attacks
- Implement secure session management

**Password Security**:
```
✅ Password Requirements
- Minimum length: 8 characters (12+ recommended)
- No maximum length (allow passphrases)
- Check against breach databases (Have I Been Pwned API)
- Enforce complexity OR length (not both - passphrases better)

✅ Password Storage
- Hash with bcrypt (work factor 12+) / argon2id / scrypt
- NEVER use MD5, SHA1, or unsalted hashes
- Salt is automatic with modern algorithms

✅ Password Reset
- Use time-limited tokens (15-30 minutes)
- Invalidate token after use
- Send reset link via email (never send password)
- Require re-authentication for sensitive changes
```

**Session Security**:
```
✅ Session Management
- Generate new session ID after login (prevent fixation)
- Expire sessions after inactivity (15-30 minutes)
- Invalidate sessions on logout (server-side)
- Use secure, httpOnly cookies for session tokens
- Regenerate session ID on privilege change

✅ Token Security (JWT/OAuth)
- Short expiration (15 minutes for access tokens)
- Store refresh tokens securely (httpOnly cookies)
- NEVER store tokens in localStorage (XSS risk)
- Include audience, issuer, expiration claims
- Use proper signature verification
```

**Brute Force Protection**:
```
✅ Rate Limiting
- Max 5 failed login attempts per 15 minutes
- Exponential backoff after failures
- Account lockout after repeated failures
- CAPTCHA after N attempts
- IP-based rate limiting
```

**Checklist**:
- [ ] Authentication via proven library/service
- [ ] Passwords hashed with modern algorithm
- [ ] MFA available for sensitive accounts
- [ ] Brute force protection implemented
- [ ] Secure session management
- [ ] Password reset flow secure
- [ ] No credentials in URLs or logs

---

### 8. Software and Data Integrity Failures

**Risk**: Code/data altered without detection (supply chain attacks, insecure CI/CD).

**Prevention Principles**:
- Verify integrity of downloads
- Use signed packages
- Implement code review
- Secure CI/CD pipelines
- Separate build and deployment environments

**Supply Chain Security**:
```
✅ Dependency Integrity
- Verify package signatures
- Use lock files (exact versions)
- Pin container image digests (not tags)
- Review dependencies before adding

✅ Build Security
- Use isolated build environments
- Sign build artifacts
- Store secrets in secret managers (not env vars in CI)
- Require code review before merge
- Automated security scans in CI
```

**Checklist**:
- [ ] Dependencies verified (checksums, signatures)
- [ ] Code review required for all changes
- [ ] Signed commits enforced
- [ ] CI/CD pipeline secured
- [ ] Build artifacts signed
- [ ] Secrets not in CI config files

---

### 9. Security Logging and Monitoring Failures

**Risk**: Breaches not detected, insufficient audit trails for forensics.

**Prevention Principles**:
- Log all security-relevant events
- Never log sensitive data
- Implement real-time monitoring
- Set up alerting for suspicious activity
- Retain logs for forensic analysis

**What to Log**:
```
✅ Security Events
- Authentication attempts (success, failure)
- Authorization failures
- Input validation failures
- Admin actions
- Data access (read sensitive records)
- Configuration changes
- Security exceptions

❌ NEVER Log
- Passwords (plaintext or hashed)
- Session tokens or API keys
- Credit card numbers
- Social security numbers
- Any PII in plaintext
```

**Log Format (Structured)**:
```json
{
  "timestamp": "2024-01-15T10:30:00Z",
  "level": "WARN",
  "event": "authentication_failure",
  "user_id": null,
  "username": "user@example.com",
  "ip": "203.0.113.42",
  "user_agent": "Mozilla/5.0...",
  "reason": "invalid_password"
}
```

**Alerting**:
```
✅ Alert On
- Multiple failed login attempts (same user or IP)
- Privilege escalation attempts
- Unusual data access patterns
- Configuration changes
- Application errors above threshold
- Unusual traffic patterns (DDoS)
```

**Checklist**:
- [ ] Security events logged
- [ ] No sensitive data in logs
- [ ] Logs centralized and searchable
- [ ] Real-time alerting configured
- [ ] Log retention policy defined
- [ ] Monitoring dashboard exists
- [ ] Failed access attempts trigger alerts

---

### 10. Server-Side Request Forgery (SSRF)

**Risk**: Attacker tricks server into making requests to internal resources.

**Prevention Principles**:
- Validate and sanitize URLs
- Use allowlists for destinations
- Disable unnecessary protocols
- Separate internal and external networks
- Implement network segmentation

**SSRF Example**:
```
❌ DANGEROUS - Arbitrary URL fetching
url = request.get_parameter("url")
response = fetch(url)
-- Attacker sends: http://localhost:6379/  # Access internal Redis
-- Attacker sends: http://169.254.169.254/  # Cloud metadata API

✅ SAFE - Allowlist + Validation
allowed_domains = ["api.example.com", "cdn.example.com"]
parsed_url = parse_url(url)

if parsed_url.hostname not in allowed_domains:
    return error("Invalid destination")

# Also block private IP ranges
if is_private_ip(parsed_url.hostname):
    return error("Access denied")

response = fetch(url)
```

**Block Private IPs**:
```
Block These Ranges:
- 127.0.0.0/8 (localhost)
- 10.0.0.0/8 (private)
- 172.16.0.0/12 (private)
- 192.168.0.0/16 (private)
- 169.254.0.0/16 (link-local)
- ::1 (IPv6 localhost)
- fc00::/7 (IPv6 private)
```

**Checklist**:
- [ ] URL validation before fetching
- [ ] Allowlist for external requests
- [ ] Private IP ranges blocked
- [ ] Unnecessary protocols disabled (file://, gopher://)
- [ ] Network segmentation in place

---

## Input Validation

**Golden Rules**:
1. **Validate on the server** (never trust client)
2. **Whitelist over blacklist** (allow known good, not block known bad)
3. **Validate type, length, format, range**
4. **Reject invalid input early**
5. **Use strong typing**

**Validation Pattern**:
```
✅ Input Validation Checklist
1. Type: Is it string/number/boolean?
2. Length: Within expected range?
3. Format: Matches expected pattern (email, UUID, etc.)?
4. Range: For numbers, within min/max?
5. Charset: Only allowed characters?
6. Business logic: Meets domain constraints?
```

**Common Inputs**:
```
Email: regex + length limit + lowercase normalization
URL: parse + protocol check + allowlist domain
Phone: digits only + length + country code validation
Date: ISO 8601 format + range check
Integer: parse + min/max range
UUID: regex ^[0-9a-f]{8}-[0-9a-f]{4}-...$
File: extension + MIME type + size + virus scan
```

**File Upload Security**:
```
✅ File Upload Validation
1. Size limit (prevent DoS)
2. File type allowlist (not blacklist)
3. Extension validation (not just MIME type)
4. Virus scanning
5. Store outside web root
6. Generate random filenames (prevent overwrites)
7. Serve via separate domain (prevent XSS)
```

---

## Output Encoding (XSS Prevention)

**Principle**: Escape output based on context.

**Contexts**:
```
1. HTML Context: < > & " '
   Use HTML entity encoding

2. JavaScript Context: </script> ' "
   Use JavaScript escaping

3. URL Context: & = ? #
   Use URL encoding

4. CSS Context: expressions, url()
   Use CSS escaping

5. SQL Context: ' " \
   Use parameterized queries (not escaping)
```

**XSS Prevention**:
```
✅ HTML Sanitization
- Use trusted libraries (DOMPurify, Bleach, etc.)
- Define allowlist of tags/attributes
- Strip all script/event handlers
- Sanitize CSS (remove expressions)

✅ Content Security Policy (CSP)
Content-Security-Policy:
  default-src 'self';
  script-src 'self';
  style-src 'self';
  img-src 'self' data: https:;
  font-src 'self';
  connect-src 'self' https://api.trusted.com;
  frame-ancestors 'none';
```

---

## Secrets Management

**Never Do**:
- ❌ Hardcode secrets in source code
- ❌ Commit secrets to version control
- ❌ Store secrets in plaintext files
- ❌ Log secrets
- ❌ Pass secrets in URLs
- ❌ Store secrets in client-side code

**Always Do**:
- ✅ Use environment variables
- ✅ Use secret management services (AWS Secrets Manager, Vault, etc.)
- ✅ Rotate secrets regularly
- ✅ Use different secrets per environment
- ✅ Encrypt secrets at rest
- ✅ Limit access to secrets (least privilege)

**Secret Scanning**:
```bash
# Scan git history for secrets
git secrets --scan

# Pre-commit hooks
pre-commit install
# Add secret scanning to pre-commit config

# CI/CD secret scanning
# Use tools like TruffleHog, GitGuardian, etc.
```

---

## API Security

**RESTful API Security**:
```
✅ Authentication & Authorization
- Require authentication for all endpoints (except public)
- Use API keys or OAuth2 tokens
- Implement rate limiting per API key
- Validate authorization on every request

✅ Input/Output
- Validate all inputs (JSON schema validation)
- Use pagination (limit max results)
- Filter output (don't leak internal fields)
- Set proper Content-Type headers

✅ HTTPS Only
- Enforce TLS 1.2+ (disable older versions)
- Use HSTS header
- No mixed content (all resources via HTTPS)

✅ CORS
- Don't use wildcard (*) in production
- Allowlist specific origins
- Require credentials for sensitive endpoints
```

**GraphQL Security**:
```
✅ Additional Considerations
- Query depth limiting (prevent nested DoS)
- Query complexity limiting
- Disable introspection in production
- Implement field-level authorization
- Rate limit by query complexity, not just requests
```

---

## Security Testing

**Types of Testing**:
1. **Unit Tests**: Test input validation, authorization logic
2. **Integration Tests**: Test authentication flows, API security
3. **Penetration Testing**: Simulate real attacks
4. **Static Analysis**: Scan code for vulnerabilities (SAST)
5. **Dynamic Analysis**: Test running application (DAST)
6. **Dependency Scanning**: Check for vulnerable dependencies (SCA)

**Test Cases to Include**:
```
✅ Authentication
- Valid credentials work
- Invalid credentials rejected
- Brute force protection active
- Session expiration works

✅ Authorization
- Users can only access their own data
- Admin actions require admin role
- Horizontal privilege escalation prevented
- Vertical privilege escalation prevented

✅ Input Validation
- Invalid input rejected (wrong type, too long, etc.)
- SQL injection attempts fail
- XSS payloads sanitized
- File uploads validated

✅ Rate Limiting
- Exceeding rate limit returns 429
- Different limits for different endpoints
- Rate limits reset after window
```

---

## Pre-Deployment Security Checklist

Before deploying to production:

### Infrastructure
- [ ] HTTPS enforced (HTTP redirects to HTTPS)
- [ ] TLS 1.2+ only (disable older protocols)
- [ ] Security headers configured
- [ ] CORS properly configured (no wildcard)
- [ ] Firewall rules reviewed
- [ ] Unnecessary ports closed
- [ ] Network segmentation in place

### Authentication & Authorization
- [ ] Strong password policy enforced
- [ ] MFA available/required
- [ ] Session management secure
- [ ] Authorization checks on all endpoints
- [ ] Default credentials changed
- [ ] Admin access restricted

### Input/Output
- [ ] All user input validated
- [ ] Output properly encoded
- [ ] File uploads validated
- [ ] SQL injection prevention verified
- [ ] XSS prevention verified

### Secrets & Data
- [ ] No secrets in source code or config files
- [ ] Secrets in environment variables / secret manager
- [ ] Sensitive data encrypted at rest
- [ ] Sensitive data encrypted in transit
- [ ] No sensitive data in logs

### Monitoring & Logging
- [ ] Security events logged
- [ ] No sensitive data in logs
- [ ] Log aggregation configured
- [ ] Alerting configured
- [ ] Error tracking enabled

### Dependencies
- [ ] All dependencies up to date
- [ ] No critical vulnerabilities (security scan passed)
- [ ] Lock files committed
- [ ] Automated security scanning enabled

### API Security
- [ ] Rate limiting enabled
- [ ] Pagination on list endpoints
- [ ] Request size limits
- [ ] Timeout limits

### Compliance (if applicable)
- [ ] GDPR: Data privacy, right to deletion
- [ ] HIPAA: PHI protected
- [ ] PCI-DSS: Credit card data handling
- [ ] SOC 2: Security controls documented

---

## Security Resources

### OWASP Resources
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/)
- [OWASP Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)

### Security Testing Tools
- Static Analysis: SonarQube, Semgrep, Bandit, ESLint security plugins
- Dependency Scanning: Snyk, Dependabot, npm audit, OWASP Dependency-Check
- Dynamic Analysis: OWASP ZAP, Burp Suite
- Secret Scanning: TruffleHog, GitGuardian, git-secrets

### Standards & Compliance
- [CWE Top 25](https://cwe.mitre.org/top25/)
- [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework)
- [PCI-DSS](https://www.pcisecuritystandards.org/)
- [GDPR](https://gdpr.eu/)

---

**Remember**:
- Security is not a feature, it's a requirement
- One vulnerability can compromise everything
- Defense in depth: multiple layers of security
- Security is an ongoing process, not a one-time task
- When in doubt, err on the side of caution
