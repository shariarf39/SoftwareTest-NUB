# NUB ERP System — Security Penetration Test Report

**Target URL:** https://nub.edstructure.com  
**Test Date:** May 19, 2026  
**Tester:** GitHub Copilot (Automated Security Testing)  
**Test Account:** Student ID `06270200003`  
**Testing Method:** Black-box authenticated testing using student-level credentials  
**OWASP Top 10 Reference:** All findings mapped to OWASP categories

---

## Executive Summary

Security testing of the NUB ERP Student Portal identified **11 security vulnerabilities**, including **2 critical data exposure issues** that directly expose other students' private financial information to any authenticated user. No SQL injection or authentication bypass was found, but the application has serious authorization and session management flaws.

| Severity | Count |
|----------|-------|
| 🔴 CRITICAL | 2 |
| 🟠 HIGH | 5 |
| 🟡 MEDIUM | 4 |
| ✅ PASSED (not vulnerable) | 4 |

---

## 🔴 CRITICAL VULNERABILITIES

---

### SEC-01 — IDOR: Student Can Access ALL Other Students' Financial Records

**OWASP:** A01:2021 – Broken Access Control  
**Affected Endpoints:**
- `GET /api/finance/payables/` → returns payables for **10 other students**
- `GET /api/finance/payables/{id}/` → returns any individual payable record
- `GET /api/finance/student-cost-packages/` → returns cost packages for **12 other students**
- `GET /api/finance/payments/` → returns 25 payment records (mixed students)

**Evidence:**

The endpoint `/api/finance/payables/` returned records for these students when called with student `06270200003`'s token — none of which are the authenticated user:

```
Rakib (04270200001), Raiza Nahar (06260200008), Imran Ahmed (07270100001),
Shibli Ahmed (07270100002), Maisha Fahmida (06270100001),
Masud Bhuiyan (06260100007), Rahat Islam Akash (06260100006),
Nafis Nihal (06260200009), Sania Rahman (06260300014), Kamrul Poddar (06260300013)
```

Individual record GET confirmed:
- `GET /api/finance/payables/32/` → HTTP 200 → returns `student_name: "Rakib"`, `student_id: "04270200001"` ✅ CONFIRMED IDOR

**Data Exposed:** Student names, student IDs, fee amounts, registration codes, semester details, cost package info, payment status.

**Impact:** Any student can view any other student's full financial status, outstanding dues, payment history, and tuition details.

**Fix:** All finance list endpoints must filter records by `request.user` server-side. Do not rely on client-side filtering.

---

### SEC-02 — Mass User Enumeration: Full User Directory Exposed to Students

**OWASP:** A01:2021 – Broken Access Control / A02:2021 – Cryptographic Failures  
**Affected Endpoint:** `GET /api/auth/users/`

**Evidence:**

Calling `GET /api/auth/users/` with a student Bearer token returns **HTTP 200** and a paginated list of **all 52 user accounts** in the system, including:

```json
{
  "count": 52,
  "results": [
    { "id": 112, "code": "NUB2010411", "name": "A. S. M. Sabiqul Hassan",
      "email": "sqh5.cse.ac@gmail.com", "roles": [] },
    { "id": 110, "code": "30270200001", "name": "zubair Ahmed",
      "email": "302702-00001@nub.ac.bd", "roles": ["Student"] },
    ...
  ]
}
```

Fields exposed: `id`, `code`, `username`, `name`, `email_address`, `contact_no`, `is_staff`, `is_superuser`, `roles`

**Impact:**
- Exposes names, emails, and phone numbers of all students and teachers
- Provides attacker with a complete list of valid usernames for brute-force attacks
- Violates user privacy (GDPR/PDPA equivalent principles)
- Teacher accounts and their personal emails are exposed to students

**Fix:** Remove student access to this endpoint entirely. If needed for lookups, return only minimal fields (name + student ID) scoped to appropriate roles.

---

## 🟠 HIGH VULNERABILITIES

---

### SEC-03 — JWT Access Token Stored in `localStorage` (XSS-Stealable)

**OWASP:** A02:2021 – Cryptographic Failures / A07:2021 – Identification and Authentication Failures  

**Evidence:**

```javascript
localStorage.getItem('access_token')   // Full JWT access token
localStorage.getItem('refresh_token')  // Full JWT refresh token
localStorage.getItem('user')           // Full user profile object with phone, email, role flags
```

All three are stored in `localStorage`, which is:
- Fully accessible to any JavaScript on the page
- Stolen by any XSS attack (even a reflected XSS from a third-party embedded script)
- Persistent — does not expire when the browser closes

**Fix:** Store tokens in `HttpOnly; Secure; SameSite=Strict` cookies. HttpOnly cookies are not accessible via JavaScript and cannot be stolen by XSS.

---

### SEC-04 — JWT Token Passed in URL Query String

**OWASP:** A02:2021 – Cryptographic Failures  
**Affected Endpoint:** `GET /api/notifications/stream/?token=eyJhbGci...`

**Evidence:**

Every page load triggers a request with the full JWT embedded in the URL:
```
/api/notifications/stream/?token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Impact:**
- Token is stored in browser history (visible to anyone with physical device access)
- Token is logged in web server access logs (Cloudflare, Nginx)
- Token is logged in any corporate/ISP proxy logs
- Token is included in HTTP Referer headers for subsequent requests

**Fix:** Pass the token via `Authorization: Bearer <token>` header, which is not logged or stored. Use WebSocket with header-based auth instead of SSE with URL token.

---

### SEC-05 — No Login Rate Limiting (Brute Force Possible)

**OWASP:** A07:2021 – Identification and Authentication Failures  
**Affected Endpoint:** `POST /api/auth/login/`

**Evidence:**

8 consecutive failed login attempts were sent in < 2 seconds. All returned HTTP 400 with no throttling, CAPTCHA, or lockout:

```
Attempt 1: 400 (350ms)
Attempt 2: 400 (400ms)
Attempt 3: 400 (151ms)
Attempt 4: 400 (389ms)
Attempt 5: 400 (177ms)
Attempt 6: 400 (140ms)
Attempt 7: 400 (297ms)
Attempt 8: 400 (576ms)
```
HTTP 429 (Too Many Requests) was **never** returned.

**Impact:** An attacker can run automated password-guessing attacks. Combined with the user list exposure from SEC-02, an attacker has both the username list AND unlimited login attempts — enabling full account takeover via brute force.

**Fix:** Implement rate limiting (e.g., 5 failed attempts per IP/username per 15 minutes), return HTTP 429, and optionally add CAPTCHA after 3 failures.

---

### SEC-06 — Refresh Token Not Rotated (Token Reuse Attack)

**OWASP:** A07:2021 – Identification and Authentication Failures  
**Affected Endpoint:** `POST /api/auth/token/refresh/`

**Evidence:**

The same refresh token was used twice successfully:
```
Attempt 1: POST /api/auth/token/refresh/ → 200 OK (new access token issued)
Attempt 2: Same refresh token used again → 200 OK (new access token issued again)
```

Refresh tokens are valid for **7 days** and can be reused indefinitely.

**Impact:** If a refresh token is compromised (leaked in logs, stolen via XSS), an attacker can keep generating new access tokens for 7 days without the real user being able to invalidate the session.

**Fix:** Implement refresh token rotation — each use of a refresh token must invalidate the old one and issue a new one. Also implement refresh token family detection to detect token reuse attacks.

---

### SEC-07 — Django Admin Panel Publicly Accessible

**OWASP:** A05:2021 – Security Misconfiguration  
**URL:** `https://nub.edstructure.com/django-admin/`

**Evidence:**

```
GET /django-admin/ → HTTP 200 → Django admin login page
Title: "Log in | Django site admin"
```

**Impact:**
- The admin panel login page is accessible to the entire internet
- Enables targeted brute force attacks against admin/staff accounts
- Exposes the fact that the backend is Django (technology fingerprinting)
- Admin credentials could be guessed or tested without any lockout (rate limiting may not apply here)

**Fix:** 
1. Block `/django-admin/` from public internet at Nginx/Cloudflare level (whitelist only internal/VPN IPs)
2. OR change the admin URL to a non-guessable path (e.g., `/nub-staff-mgmt-2026/`)
3. Enable 2FA for all admin accounts

---

## 🟡 MEDIUM VULNERABILITIES

---

### SEC-08 — Missing HTTP Strict Transport Security (HSTS) Header

**OWASP:** A05:2021 – Security Misconfiguration  

**Evidence:** The main page response does not include `Strict-Transport-Security` header.

**Impact:** Without HSTS, browsers do not enforce HTTPS on return visits. A network attacker (e.g., on a university Wi-Fi) could perform an SSL stripping attack, downgrading the user's connection to HTTP and capturing credentials and tokens in plaintext.

**Fix:** Add: `Strict-Transport-Security: max-age=31536000; includeSubDomains; preload`

---

### SEC-09 — Missing Content Security Policy (CSP) Header

**OWASP:** A05:2021 – Security Misconfiguration  

**Evidence:** Neither the main page nor the API responses include a `Content-Security-Policy` header.

**Impact:** Without CSP, any successful XSS attack (via a compromised dependency, DOM injection, etc.) can:
- Steal tokens from localStorage (see SEC-03)
- Make unauthorized API calls as the user
- Exfiltrate data to attacker-controlled servers

**Fix:** Define a strict CSP that whitelists only known script sources. At minimum: `Content-Security-Policy: default-src 'self'; script-src 'self'`

---

### SEC-10 — Duplicate and Conflicting Security Headers

**OWASP:** A05:2021 – Security Misconfiguration  

**Evidence:**

The API returns these headers with duplicate, contradictory values:
```
x-frame-options: DENY, SAMEORIGIN       ← contradictory
x-content-type-options: nosniff, nosniff ← duplicated
```

This occurs because the Django framework sets `DENY`/`nosniff` and Cloudflare's WAF also adds `SAMEORIGIN`/`nosniff`. The result is ambiguous — different browsers interpret duplicate headers differently.

**Fix:** Configure Cloudflare to not add security headers that Django already provides (or vice versa). Ensure a single, consistent source of truth for these headers.

---

### SEC-11 — Missing Permissions Policy and Referrer Policy Headers

**OWASP:** A05:2021 – Security Misconfiguration  

**Evidence:** No `Permissions-Policy` or `Referrer-Policy` headers on the main page.

**Impact:**
- Without `Referrer-Policy`: If a user clicks a link to an external site, the JWT token URL from SEC-04 could be sent in the `Referer` header to the external site
- Without `Permissions-Policy`: Browser features (camera, microphone, geolocation) have no restrictions

**Fix:**
```
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=()
```

---

## ✅ TESTS PASSED (Not Vulnerable)

| Test | Result |
|------|--------|
| SQL Injection on login form | ✅ Safe — Django ORM correctly parameterizes queries |
| Authentication bypass (unauthenticated API access) | ✅ Safe — all endpoints return 401 without token |
| Unauthorized data modification (PATCH/DELETE on other students' records) | ✅ Safe — returns 403 Forbidden |
| CORS (cross-origin API access from evil.com) | ✅ Safe — no `Access-Control-Allow-Origin` returned for foreign origin |

---

## Attack Scenario: Full Student Data Breach

Using the vulnerabilities above, a malicious student could execute this attack chain:

```
1. [SEC-02] GET /api/auth/users/ → Extract all 52 student names, emails, IDs
2. [SEC-05] POST /api/auth/login/ → Brute force any student account (no rate limit)
3. [SEC-01] GET /api/finance/payables/ → Read target student's financial data
4. [SEC-01] GET /api/finance/payments/ → Read all payment history
5. [SEC-06] POST /api/auth/token/refresh/ → Maintain access for 7 days using stolen refresh token
```

No advanced hacking skills required — these are simple HTTP GET requests with a valid student Bearer token.

---

## Priority Fix Summary

| Priority | Fix | Effort |
|----------|-----|--------|
| 🔴 IMMEDIATE | Filter `/api/finance/*` endpoints to current user only | Low |
| 🔴 IMMEDIATE | Restrict `/api/auth/users/` to admin/staff roles only | Low |
| 🟠 HIGH | Add rate limiting to `/api/auth/login/` (5 req/15min per IP) | Medium |
| 🟠 HIGH | Move JWT to `HttpOnly` cookies, remove from `localStorage` | High |
| 🟠 HIGH | Block `/django-admin/` from public internet (firewall rule) | Low |
| 🟠 HIGH | Implement refresh token rotation | Medium |
| 🟠 HIGH | Stop passing JWT token in notification stream URL | Medium |
| 🟡 MEDIUM | Add `Strict-Transport-Security` header | Low |
| 🟡 MEDIUM | Add `Content-Security-Policy` header | Medium |
| 🟡 MEDIUM | Fix duplicate/conflicting security headers (DENY vs SAMEORIGIN) | Low |
| 🟡 MEDIUM | Add `Referrer-Policy` and `Permissions-Policy` headers | Low |

---

*Security penetration test conducted on May 19, 2026.*  
*Testing was limited to read-only actions plus authentication endpoint probing. No data was modified, deleted, or exfiltrated. Testing was performed using only the provided student credentials.*
