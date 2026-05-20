# NUB ERP — Post-Delivery Software Testing Plan

**Version:** 1.0  
**Date:** May 20, 2026  
**Purpose:** This document defines the complete testing procedure to run after every major release or deployment of the NUB ERP system, ensuring quality, security, and reliability before handing over to the client.

---

## Overview

Testing is organized into 5 phases:

| Phase | Type | When |
|-------|------|------|
| Phase 1 | Smoke Tests | Immediately after deployment |
| Phase 2 | Functional Tests | Before client handover |
| Phase 3 | Security Tests | Before every release |
| Phase 4 | Performance Tests | Before every major release |
| Phase 5 | Regression Tests | After every bug fix or change |

---

## Phase 1: Smoke Tests (Run First — ~15 minutes)

Quick checks to confirm the system is alive and basic functionality works.

### 1.1 Server Availability
- [ ] Home page (`/`) loads with HTTP 200
- [ ] Login page (`/login`) loads correctly
- [ ] API health endpoint (`/health/`) returns "OK"
- [ ] No JavaScript errors in browser console on page load
- [ ] Static assets (CSS, JS) all load (no 404 in network tab)

### 1.2 Authentication Flow
- [ ] Student can log in with valid credentials
- [ ] Invalid credentials return HTTP 400 with error message (not 500)
- [ ] After login, JWT token is issued
- [ ] After logout, token is invalidated
- [ ] Expired token returns 401 (not 500)

### 1.3 Core Pages Load
- [ ] Student dashboard (`/student`) loads after login
- [ ] Profile page shows correct student name
- [ ] Navigation menu renders without errors
- [ ] No broken images or links on the main dashboard

---

## Phase 2: Functional Tests (~2–3 hours)

Verify every feature works as expected.

### 2.1 Authentication & User Management
- [ ] Login with student account → redirects to `/student`
- [ ] Login with teacher account → redirects to teacher dashboard
- [ ] Login with admin account → redirects to admin dashboard
- [ ] Forgot password flow sends email (if implemented)
- [ ] Session expires after inactivity timeout
- [ ] Concurrent login from two devices behaves correctly

### 2.2 Student Portal
- [ ] Student profile displays correct: name, ID, program, department
- [ ] Photo displays correctly
- [ ] Personal info update (if allowed) saves and reflects immediately
- [ ] Semester/program info is accurate
- [ ] GPA/CGPA displays correctly

### 2.3 Academic Module
- [ ] Course list shows correct courses for student's semester
- [ ] Class schedule displays correctly (days, times, rooms)
- [ ] Attendance percentage shows correctly
- [ ] Course registration form (if applicable) works
- [ ] Academic calendar shows correct dates

### 2.4 Examination Module
- [ ] Exam schedule shows correct dates and courses
- [ ] Seating plan shows correct room and seat number
- [ ] Exam results display after publication
- [ ] Grade report generates PDF correctly
- [ ] Transcript download works
- [ ] Result history shows all semesters

### 2.5 Finance Module
- [ ] Student sees ONLY their own payables (not other students')
- [ ] Payment history shows all transactions
- [ ] Fee breakdown (payable-details) is correct
- [ ] Invoice/receipt PDF generates correctly
- [ ] Due balance is accurately calculated
- [ ] Late fee appears after due date

### 2.6 Notification System
- [ ] Student receives notifications
- [ ] Read/unread status toggles correctly
- [ ] Notification count badge updates
- [ ] Real-time notification stream connects (WebSocket/SSE)
- [ ] System does not crash (no HTTP 500) on notification fetch

### 2.7 HR Module (Admin/Staff access only)
- [ ] Admin can list all employees
- [ ] Admin can create new employee record
- [ ] Admin can update employee salary
- [ ] Admin can record leave applications
- [ ] Payroll report generates correctly
- [ ] **Student account CANNOT access any HR endpoint** (must return 403)

### 2.8 Admission Module
- [ ] Admin can create student admission record
- [ ] Student can view ONLY their own admission record
- [ ] **Student CANNOT see other students' admission records**
- [ ] Document upload works correctly
- [ ] Admission status updates correctly

### 2.9 Guardian Portal
- [ ] Guardian can log in with guardian account
- [ ] Guardian sees only their ward's data
- [ ] Guardian cannot see other students' data

---

## Phase 3: Security Tests (~3–4 hours)

**Run every test in this phase before any production release.**

### 3.1 Authentication Security

| Test | Expected Result |
|------|-----------------|
| Login with wrong password | HTTP 400 — "Invalid credentials" |
| Login 10 times wrong → | HTTP 429 — Rate limited |
| Login with SQL injection (`' OR '1'='1`) | HTTP 400 — No database error |
| Login with empty username | HTTP 400 — Validation error |
| Use expired JWT token | HTTP 401 — Token expired |
| Use tampered JWT (changed user_id) | HTTP 401 — Invalid signature |
| Use JWT with `"alg":"none"` | HTTP 401 — Rejected |
| Send requests with no token | HTTP 401 — Unauthorized |
| Brute-force token (rapid requests) | HTTP 429 after threshold |

### 3.2 Broken Access Control (CRITICAL — Test Every Endpoint)

For each major endpoint, log in as **Student A** and test:

#### Student isolation — Student A must NOT see Student B's data:
- [ ] `GET /api/admission/students/` → returns **only own record** (count=1)
- [ ] `GET /api/finance/payables/` → returns **only own payables**
- [ ] `GET /api/finance/payments/` → returns **only own payments**
- [ ] `GET /api/examination/exam-results/` → returns **only own results**
- [ ] `GET /api/notifications/notifications/` → returns **only own notifications**

#### HR module must be completely blocked for students:
- [ ] `GET /api/hr/employees/` → must return **403 Forbidden** (not 200)
- [ ] `GET /api/hr/employee-salaries/` → must return **403 Forbidden**
- [ ] `GET /api/hr/employee-bank-accounts/` → must return **403 Forbidden**
- [ ] `GET /api/hr/provident-funds/` → must return **403 Forbidden**
- [ ] `GET /api/hr/leave-applications/` → must return **403 Forbidden**
- [ ] `POST /api/hr/employees/` → must return **403 Forbidden**
- [ ] `DELETE /api/hr/leave-applications/1/` → must return **403 Forbidden**

#### Audit logs must be restricted:
- [ ] `GET /api/notifications/access-logs/` → **403** for students
- [ ] `GET /api/notifications/activity-logs/` → **403** for students
- [ ] `GET /api/notifications/password-change-logs/` → **403** for students
- [ ] `GET /api/notifications/result-change-logs/` → **403** for students

#### RBAC map must be restricted:
- [ ] `GET /api/auth/user-roles/` → **403** for students (no other users' roles visible)
- [ ] `GET /api/auth/role-tasks/` → **403** for students
- [ ] `GET /api/auth/users/` → **403** for students (or only own profile)

### 3.3 Injection Tests

| Test | Endpoint | Input | Expected |
|------|----------|-------|----------|
| SQL injection | `/api/auth/login/` | `"username": "' OR 1=1 --"` | 400, no DB error |
| XSS in name | `/api/auth/profile/` | `"name": "<script>alert(1)</script>"` | Stored escaped, not executed |
| XSS in notice | Any text field | `<img src=x onerror=alert(1)>` | Rendered as text, not HTML |
| Path traversal | Any file parameter | `../../etc/passwd` | 400 or 404, no file content |
| Command injection | Any input | `; ls -la` | No command output in response |

### 3.4 Media & File Security

- [ ] `/media/` files require authentication to access (not public)
- [ ] Only the owner can view their own documents
- [ ] File upload (if present) only accepts allowed types (jpg, png, pdf)
- [ ] Uploaded files are not served as executable scripts
- [ ] File size limit is enforced

### 3.5 Hardcoded Credentials Check

- [ ] Search the JS bundle for `password=`, `api_key=`, `secret=`, `token=`
- [ ] No test credentials (like `123456`, `admin`, `password`) exist in any account
- [ ] No API keys or secrets are present in frontend code
- [ ] No `.env` file is committed to the repository
- [ ] No database credentials appear in JavaScript or HTML

### 3.6 HTTP Security Headers

Verify ALL headers are present in every response:

| Header | Required Value |
|--------|----------------|
| `Content-Security-Policy` | Must NOT contain `'unsafe-inline'` or `'unsafe-eval'` |
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` |
| `X-Frame-Options` | `DENY` |
| `X-Content-Type-Options` | `nosniff` |
| `Referrer-Policy` | `strict-origin-when-cross-origin` |
| `Permissions-Policy` | Defined and restrictive |

### 3.7 Session Management

- [ ] JWT access token expiry is ≤ 60 minutes
- [ ] Refresh token is rotated on every use (old token invalidated)
- [ ] Logout invalidates the refresh token server-side
- [ ] JWT is stored in HttpOnly cookie (NOT localStorage)
- [ ] No token appears in URL parameters

---

## Phase 4: Performance Tests (~1–2 hours)

### 4.1 Load Testing Baselines

Use a tool like Apache JMeter or k6:

| Scenario | Target | Acceptable |
|----------|--------|------------|
| 50 concurrent students loading dashboard | < 2 seconds | < 5 seconds |
| 100 concurrent login requests | All succeed | No 500 errors |
| Large list API (1000 items) | < 3 seconds | < 8 seconds |
| PDF generation (transcript) | < 5 seconds | < 15 seconds |

### 4.2 Page Load Times (Browser)

Using Chrome DevTools or Lighthouse:
- [ ] First Contentful Paint (FCP) < 3 seconds
- [ ] Time to Interactive (TTI) < 5 seconds
- [ ] Largest Contentful Paint (LCP) < 4 seconds
- [ ] Cumulative Layout Shift (CLS) < 0.1
- [ ] JavaScript bundle total size < 3 MB (currently 5 MB — needs optimization)

### 4.3 Database Queries

- [ ] No N+1 query problems on list endpoints
- [ ] Pagination works correctly (page size ≤ 100 records per request)
- [ ] Filter/search does not cause full table scans

---

## Phase 5: Regression Tests

Run these after every bug fix or code change:

### 5.1 Bug Regression Checklist

After fixing any bug from the Bug Report, verify:
- [ ] The specific bug no longer occurs
- [ ] The fix did not break related features
- [ ] Edge cases around the fix are tested

### 5.2 API Contract Testing

After any API change:
- [ ] All existing API endpoints still return the same field names
- [ ] No endpoint that was previously 200 now returns 404
- [ ] No new required fields break existing client code
- [ ] Pagination structure (`count`, `next`, `previous`, `results`) is consistent

### 5.3 Cross-Browser Testing

- [ ] Chrome (latest)
- [ ] Firefox (latest)
- [ ] Edge (latest)
- [ ] Mobile Chrome (Android)
- [ ] Safari (iOS) — if mobile support is required

---

## Test Accounts Required

The following accounts must exist in every test environment:

| Account Type | Username | Purpose |
|-------------|---------|---------|
| Student A | test_student_01 | Primary test student |
| Student B | test_student_02 | Cross-account IDOR tests |
| Teacher | test_teacher_01 | Teacher feature tests |
| HR Staff | test_hr_01 | HR module tests |
| Admin | test_admin_01 | Admin panel tests |
| Guardian | test_guardian_01 | Guardian portal tests |

**Rules for test accounts:**
- Passwords must be strong and unique (not `123456`)
- Test accounts must NOT exist in production
- Test accounts must be created via server-side scripts, not hardcoded in frontend

---

## Automated Test Script (Quick Security Check)

Save this as a curl script to run after every deployment:

```bash
#!/bin/bash
# quick-security-check.sh
# Usage: ./quick-security-check.sh <base_url> <student_token>

BASE=$1
TOKEN=$2

echo "=== NUB ERP Quick Security Check ==="

# Check HR module is blocked
echo -n "HR employees blocked for student: "
STATUS=$(curl -s -o /dev/null -w "%{http_code}" -H "Authorization: Bearer $TOKEN" $BASE/api/hr/employees/)
[ "$STATUS" = "403" ] && echo "PASS" || echo "FAIL ($STATUS)"

# Check admission list is filtered
echo -n "Admission list filtered: "
COUNT=$(curl -s -H "Authorization: Bearer $TOKEN" $BASE/api/admission/students/ | python3 -c "import sys,json; print(json.load(sys.stdin)['count'])")
[ "$COUNT" = "1" ] && echo "PASS" || echo "FAIL (count=$COUNT, should be 1)"

# Check finance IDOR
echo -n "Finance payables filtered: "
COUNT=$(curl -s -H "Authorization: Bearer $TOKEN" $BASE/api/finance/payables/ | python3 -c "import sys,json; print(json.load(sys.stdin)['count'])")
[ "$COUNT" = "1" ] && echo "PASS (own record only)" || echo "WARN: count=$COUNT (check if all are own records)"

# Check access logs blocked
echo -n "Access logs blocked for student: "
STATUS=$(curl -s -o /dev/null -w "%{http_code}" -H "Authorization: Bearer $TOKEN" $BASE/api/notifications/access-logs/)
[ "$STATUS" = "403" ] && echo "PASS" || echo "FAIL ($STATUS)"

# Check login rate limiting
echo -n "Login rate limiting active: "
for i in {1..15}; do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" -X POST -H "Content-Type: application/json" \
    -d '{"username":"wrong","password":"wrong"}' $BASE/api/auth/login/)
  [ "$STATUS" = "429" ] && { echo "PASS (blocked at attempt $i)"; break; }
done

echo "=== Check complete ==="
```

---

## Sign-Off Checklist Before Delivery

All items below must be checked before handing the system to the client:

- [ ] Phase 1 Smoke Tests — all passed
- [ ] Phase 2 Functional Tests — all passed or documented exceptions
- [ ] Phase 3 Security Tests — ALL P0 items passed with no exceptions
- [ ] Phase 4 Performance — page load < 5 seconds confirmed
- [ ] No hardcoded passwords in JavaScript bundle
- [ ] No test accounts with weak passwords (`123456`, `admin`, `password`)
- [ ] All previously reported bugs from Bug Report are fixed and verified
- [ ] All previously reported security issues from Security Reports are fixed
- [ ] HTTP security headers are all present
- [ ] Student cannot access HR, audit logs, or other students' data
- [ ] Django admin is restricted to authorized IPs or VPN
- [ ] Database backup confirmed and restorable
- [ ] SSL certificate valid (check expiry date)
- [ ] Error pages return generic messages (no stack traces in production)

---

*This testing plan is based on vulnerabilities and bugs discovered during security testing on May 19–20, 2026. Update this document whenever new features are added or new vulnerabilities are discovered.*
