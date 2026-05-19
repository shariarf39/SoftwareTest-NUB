# NUB ERP System — Software Testing Report

**URL:** https://nub.edstructure.com/login  
**Test Date:** May 19, 2026  
**Tester:** GitHub Copilot (Automated Browser Testing)  
**Account Tested:** Student ID `06270200003` (Student: Shahariar)  
**Platform:** Web (Desktop Browser)

---

## Executive Summary

A total of **21 issues** were identified across the NUB ERP Student Portal, ranging from **critical functional bugs** (broken routes, non-functional features) to **UI/UX problems** (raw error messages, inconsistent labels, missing values) and **content errors** (typos, data inconsistencies).

| Severity | Count |
|----------|-------|
| 🔴 Critical | 5 |
| 🟠 High | 7 |
| 🟡 Medium | 6 |
| 🔵 Low | 3 |

---

## 🔴 CRITICAL BUGS

---

### BUG-01 — Broken Link: Dashboard "CGPA" Card Routes to Wrong Page
- **Location:** Dashboard → CGPA card (links to `/student/results/transcript`)
- **Expected:** Opens the Academic Transcript / Results page
- **Actual:** Navigates to the **TER Evaluation** page instead
- **Impact:** Students cannot access their transcript from the dashboard shortcut
- **Steps to Reproduce:**
  1. Log in and go to Dashboard
  2. Click the "CGPA 0.00" card

---

### BUG-02 — Broken Link: "Outstanding Dues" Card Routes to Dashboard
- **Location:** Dashboard → Outstanding Dues card (links to `/student/finance/dues`)
- **Expected:** Opens a Finance/Dues detail page
- **Actual:** Redirects silently to the **Dashboard** (`/student`) — the route does not exist
- **Impact:** Students cannot view their payment details from the dashboard
- **Screenshot Evidence:** URL changes to `/student` after clicking

---

### BUG-03 — Two Broken Quick Action Links on Dashboard
- **Location:** Dashboard → Quick Actions section
  - "View Class Routine" → `/student/academic/routine` → redirects to `/student`
  - "Credits Completed" link → `/student/academic/schedule` → redirects to `/student`
- **Expected:** Navigate to Class Routine and Semester Schedule pages respectively
- **Actual:** Both silently redirect to the Dashboard
- **Impact:** Quick Action shortcuts are completely non-functional

---

### BUG-04 — Notifications Streaming Endpoint Always Fails (HTTP 406)
- **Location:** Every page load — `/api/notifications/stream/?token=...`
- **Expected:** Real-time notification stream works
- **Actual:** Server returns **HTTP 406 (Not Acceptable)** on every page, every time
- **Console Error:** `Failed to load resource: the server responded with a status of 406`
- **Impact:** Real-time notifications are completely broken for all students

---

### BUG-05 — "Forgot Password?" Link is Dead (Non-functional)
- **Location:** Login page → "Forgot password?" link
- **Expected:** Opens password recovery flow
- **Actual:** Link href is `#` — nothing happens when clicked. No modal, no redirect.
- **Impact:** Students who forget their password have no self-service recovery option

---

## 🟠 HIGH BUGS

---

### BUG-06 — Raw API Error Exposed on Login Failure
- **Location:** Login page → failed login attempt
- **Expected:** User-friendly message like "Invalid username or password. Please try again."
- **Actual:** Displays raw technical error: **"Request failed with status code 400"**
- **Impact:** Poor user experience; leaks implementation details to end users

---

### BUG-07 — Raw Enum Value Displayed in Dashboard
- **Location:** Dashboard → Academic Summary → Registration Status
- **Expected:** Human-readable text like "Not Registered"
- **Actual:** Displays raw database enum value: **"NOT_REGISTERED"** (with underscore)
- **Impact:** Looks unprofessional; confusing to non-technical students

---

### BUG-08 — Incorrect "Current Semester" (Wrong Year)
- **Location:** Dashboard, Class Routine, Semester Schedule, Add/Drop, Section Change
- **Expected:** Current semester should match the current academic year (2026)
- **Actual:** **"Summer 2027"** is shown as the current semester everywhere (a full year ahead)
- **Impact:** All semester-related data is misaligned; students see wrong semester information
- **Note:** The semester period dates are April 7–8, 2026, yet the label says "2027"

---

### BUG-09 — Semester Period Duration is 1 Day (Data Error)
- **Location:** Semester Schedule page → Semester Period card
- **Expected:** A semester should span several months (e.g., 4–6 months)
- **Actual:** Semester Period: **Start: April 7th, 2026 → End: April 8th, 2026** (1 day only)
- **Impact:** Incorrect academic calendar displayed to all students

---

### BUG-10 — Credit Count Mismatch Between Dashboard and Curriculum
- **Location:** 
  - Dashboard card: "Credits Completed **0 / 30**"
  - Curriculum page: "Total Required: **42**"
- **Expected:** Both should show the same total credit requirement
- **Actual:** Dashboard says 30, Curriculum says 42 — **12 credit discrepancy**
- **Impact:** Students receive conflicting information about graduation requirements

---

### BUG-11 — "Add/Drop Period" Shows "Dates not set" but Status is "Open"
- **Location:** Add/Drop Courses page
- **Expected:** If dates are not configured, the period should show status "Not Configured" or "Pending"
- **Actual:** Status badge shows **"Open"** while subtitle reads **"Dates not set"** — contradictory
- **Impact:** Students may believe add/drop is available when nothing is actually configured

---

### BUG-12 — JWT Token Exposed in URL Query String
- **Location:** Network request to `/api/notifications/stream/?token=eyJhbGci...`
- **Expected:** Token passed via Authorization header
- **Actual:** Full JWT access token passed as a URL query parameter
- **Impact (Security):** URL query parameters are stored in browser history, server logs, and proxy logs — token is potentially exposed to unauthorized parties
- **Risk Level:** Medium-High (OWASP: Sensitive Data Exposure)

---

## 🟡 MEDIUM BUGS

---

### BUG-13 — Misleading "Low CGPA Alert" for Unregistered Students
- **Location:** Dashboard → Notifications & Alerts section
- **Expected:** Low CGPA alert should only trigger for students who have completed at least one semester
- **Actual:** Student with CGPA 0.00 (never registered/0 courses taken) receives: **"Your CGPA is below 2.5. Consider meeting with your advisor..."**
- **Impact:** Confuses/alarms new students who have not yet taken any courses

---

### BUG-14 — Sidebar Label "Notices" but Page Title Says "Notifications"
- **Location:** Sidebar nav → "Notices" link → `/student/notices`
- **Expected:** Page heading matches sidebar label
- **Actual:** Sidebar says **"Notices"**; page heading says **"Notifications"**
- **Impact:** Label inconsistency reduces trust and navigation clarity

---

### BUG-15 — "Drop Allowed" Badge Shown When Student Has 0 Registered Courses
- **Location:** Add/Drop Courses page → header badge
- **Expected:** "Drop Allowed" should only appear when the student has at least one registered course to drop
- **Actual:** Shows **"Drop Allowed"** badge when Registered Courses = **0**
- **Impact:** Misleads students; they have nothing to drop

---

### BUG-16 — Course Materials: "Total Size" Shows No Value
- **Location:** Course Materials page → "Total Size" stat card
- **Expected:** Should display "0 MB", "0 KB", or "N/A" even when no materials exist
- **Actual:** The card shows only the label **"Total Size"** with **no value at all** (empty)
- **Impact:** UI looks broken/incomplete

---

### BUG-17 — Profile: Gender Field Not Populated
- **Location:** My Profile → Gender field
- **Expected:** A filled or required field value
- **Actual:** Shows **"Select Gender"** placeholder — the field was never filled
- **Impact:** Incomplete student profile data; may affect records

---

### BUG-18 — Convocation Registration Page Missing Description Subtitle
- **Location:** Convocation Registration page header
- **Expected:** Every page has a subtitle (e.g., "View your scholarship and waiver details")
- **Actual:** Convocation Registration page has a heading but **no description subtitle**, and below the single message the page is completely empty — lots of wasted blank space
- **Impact:** UI inconsistency; sparse page feels unfinished

---

## 🔵 LOW / COSMETIC BUGS

---

### BUG-19 — Typo in Course Name: "Litertature"
- **Location:** Curriculum page → Semester 1 → Course BANG 5111
- **Actual:** "History of Bengali **Litertature**-1"
- **Expected:** "History of Bengali **Literature**-1"

---

### BUG-20 — Typo in Course Name: "Morpholigical"
- **Location:** Curriculum page → Semester 3 → Course BANG 5131
- **Actual:** "**Morpholigical** Theory of Literature"
- **Expected:** "**Morphological** Theory of Literature"

---

### BUG-21 — Inconsistent Capitalization in Course Name
- **Location:** Curriculum page → Semester 1 → Course BANG 5113
- **Actual:** "**poetry** of Ancient and Medieval Age" (lowercase first letter)
- **Expected:** "**Poetry** of Ancient and Medieval Age"

---

## Additional Observations

| Observation | Detail |
|---|---|
| Timestamp anomaly in logs | Multiple network requests logged with timestamp `1969-12-31T23:59:59.999Z` (Unix epoch -1ms). Indicates a timestamp/logging bug in CDN RUM module. |
| NID field shows suspicious test data | Profile shows NID: `12345693658741555` (17 digits — Bangladesh NID is 10 or 13 digits). Looks like fake/test data. |
| "Class Routine" stat card inconsistency | "Total Classes" shows `—` (dash) while all other stat cards show `0`. Inconsistent null representation. |
| Semester Drop accessible with 0 courses | A student with no registered courses can access and submit a "Semester Drop Application" without any validation guard. |
| Profile dropdown shows student ID instead of name | Header user button displays `06270200003` instead of the student's name "Shahariar" — less personal/friendly. |

---

## Summary Table

| ID | Issue | Severity | Page |
|----|-------|----------|------|
| BUG-01 | CGPA card links to TER Evaluation instead of Transcript | 🔴 Critical | Dashboard |
| BUG-02 | Outstanding Dues card redirects to dashboard | 🔴 Critical | Dashboard |
| BUG-03 | Both Quick Action links redirect to dashboard | 🔴 Critical | Dashboard |
| BUG-04 | Notifications streaming broken (HTTP 406) | 🔴 Critical | All pages |
| BUG-05 | "Forgot Password?" link is dead | 🔴 Critical | Login |
| BUG-06 | Raw API error shown on bad login | 🟠 High | Login |
| BUG-07 | Raw enum "NOT_REGISTERED" displayed | 🟠 High | Dashboard |
| BUG-08 | Wrong current semester year (2027 vs 2026) | 🟠 High | Multiple |
| BUG-09 | Semester period is only 1 day long | 🟠 High | Semester Schedule |
| BUG-10 | Credit total mismatch (30 vs 42) | 🟠 High | Dashboard / Curriculum |
| BUG-11 | Add/Drop "Dates not set" but shows "Open" | 🟠 High | Add/Drop |
| BUG-12 | JWT token exposed in URL | 🟠 High | All pages (Network) |
| BUG-13 | Low CGPA alert fires for new students | 🟡 Medium | Dashboard |
| BUG-14 | "Notices" label vs "Notifications" heading | 🟡 Medium | Notices page |
| BUG-15 | "Drop Allowed" shown with 0 courses | 🟡 Medium | Add/Drop |
| BUG-16 | "Total Size" card has no value | 🟡 Medium | Course Materials |
| BUG-17 | Gender field blank in profile | 🟡 Medium | Profile |
| BUG-18 | Convocation page missing subtitle | 🟡 Medium | Convocation Reg. |
| BUG-19 | Typo: "Litertature" | 🔵 Low | Curriculum |
| BUG-20 | Typo: "Morpholigical" | 🔵 Low | Curriculum |
| BUG-21 | Lowercase course name "poetry" | 🔵 Low | Curriculum |

---

*Report generated by automated browser testing on May 19, 2026.*
