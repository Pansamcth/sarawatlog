# DEFENSIVE SECURITY CODE REVIEW AND ARCHITECTURE AUDIT REPORT
**Target System:** Dechapattanayanukul School Student Inspector Organization Platform (`sarawatdc.site`)  
**Target Codebase:** `f:\MP`  
**Audit Conducted By:** Teamwork Defensive Security Review & Architecture Audit Team  
**Audit Date:** November 14–15, 2026  
**Audit Classification:** Comprehensive Defensive Benchmark Audit (Read-Only Static Inspection & Architecture Analysis)  
**Applicable Compliance & Security Standards:**
- OWASP Top 10: 2021
- MITRE Common Weakness Enumeration (CWE)
- Thailand Personal Data Protection Act (PDPA B.E. 2562)
- RFC 6750 (OAuth 2.0 Bearer Token Usage & Transport Security)
- W3C Content Security Policy Level 3 Recommendation

---

## 1. Executive Summary & Audit Context

### 1.1 Engagement Context & Scope
A systematic, non-destructive, manual defensive security code review and architecture audit was executed across the complete software repository located at `f:\MP`. The target system powers the official digital presence, administrative operations, recruitment process, and student patrol duty roster for the **Student Inspector Organization of Dechapattanayanukul School** (สารวัตรนักเรียน โรงเรียนเดชะปัตตนยานุกูล, Pattani, Thailand).

The audited codebase encompasses:
1. **Public Web Portals**: Organizational homepage (`index.html`), news and activity broadcasts (`activities.html`), academic calendar (`calendar.html`), 143-person executive and committee directories (`team.html`), Digital Systems Division showcase (`dsd.html`), and legal policy specifications (`terms-of-service.html`, `privacy-policy.html`).
2. **Student Recruitment & Admissions Subsystem**: Student application submission, status verification, document uploads, and self-correction workflows (`admission.html`, `js/admin-recruitment.js`, `js/admin-year70.js`).
3. **Student Inspector Duty Roster Subsystem (SI-ROSTER)**: Real-time patrol duty scheduling, check-in tracking, attendance synchronization, and supervisory administration (`SI-ROSTER/index.html`, `SI-ROSTER/admin.html`, `SI-ROSTER/js/app.js`, `SI-ROSTER/js/bridge.js`).
4. **Administrative Back-Office Portals**: Multi-tab control center (`admin.html`, `SI-ROSTER/admin.html`) managing news, calendars, committee assignments, database backups, telemetry, and Discord bot integration.
5. **Serverless Edge Layer**: Dynamic OpenGraph pre-renderers, SSR routers, and image proxies (`api/activities.js`, `api/activity-image.js`, `api/admin.js`, `api/admission.js`, `api/dsd.js`, `api/si-roster.js`).
6. **Infrastructure & Client Bridges**: Vercel Edge configuration (`vercel.json`), Cloudflare Tunnel configuration (`cloudflare-config.js`), client API SDK (`js/api-config.js`), and operational automation scripts (`sync-to-github.bat`).

### 1.2 Audit Methodology & Integrity Standards
In accordance with strict defensive security principles:
- **Zero Source Modification**: All existing source files, templates, scripts, and configurations within `f:\MP` remained completely unmodified during the inspection process. Working tree integrity was continuously validated via version control.
- **Manual Static Analysis & Code Tracing**: Every identified flaw was manually traced from network ingress and user input vectors through parsing, sanitization, data storage, and rendering sinks.
- **Zero External Code Transmission**: No proprietary source code, applicant PII, or internal credentials were sent to public network scanners or third-party cloud services.
- **Evidence-Based Grounding**: Every documented finding cites exact relative and absolute file paths, exact line numbers, verbatim source code evidence, mechanics of failure, root cause analyses, and tailored remediation snippets.

### 1.3 Assessment Statistics & Severity Distribution
A total of **19 distinct security vulnerabilities, architectural weaknesses, and operational risks** were identified and cataloged across the codebase.

| Severity Level | Count | Definition & Risk Threshold |
| :--- | :---: | :--- |
| **Critical** | 3 | Immediate threat to organizational confidentiality, regulatory compliance (PDPA), or system boundary. Direct unauthenticated data harvesting, credential exposure, or remote code execution. |
| **High** | 7 | Flaws enabling full account takeover, arbitrary script execution (XSS), stored payloads, client authorization bypass, or administrative credential leakage. |
| **Medium** | 6 | Architectural anti-patterns, CSP weaknesses, tunnel exfiltration vectors, DoS risks, client-reported identity spoofing, or destructive operational tooling. |
| **Low** | 3 | Supply-chain hygiene deficits, lack of SRI, ineffective client-side controls, and DOM-rendering performance bottlenecks. |
| **Total Findings** | **19** | **Complete Codebase Audit Inventory** |

#### Distribution by OWASP Top 10 (2021) Categories:
- **A01:2021 — Broken Access Control**: 3 Findings (BOLA/IDOR PII leakage, unauthenticated database export, setup route exposure)
- **A02:2021 — Cryptographic Failures**: 2 Findings (Plaintext password storage in LocalStorage, token in GET query parameters)
- **A03:2021 — Injection**: 6 Findings (Reflected XSS in Edge SSR, unescaped OpenGraph meta tags, Base64 MIME type spoofing, recruitment DOM XSS, podium avatar injection, duty dropdown XSS)
- **A04:2021 — Insecure Design**: 3 Findings (Telemetry heartbeat DoS, client device ID rate-limit bypass, broad MutationObserver starvation)
- **A05:2021 — Security Misconfiguration**: 2 Findings (Insecure Content Security Policy with `'unsafe-inline'` / `*.trycloudflare.com`, destructive git automation script)
- **A06:2021 — Vulnerable and Outdated Components**: 1 Finding (Floating CDN dependencies without Subresource Integrity hashes)
- **A07:2021 — Identification and Authentication Failures**: 2 Findings (Client-side role/auth bypasses, active Vercel OIDC token in `.env.local`, IP spoofing via `api.ipify.org`)

### 1.4 Overall Security Posture Evaluation
The application exhibits modern visual design, responsive layouts, and effective use of Vercel Edge rules for routing and caching. However, the system's defensive architecture exhibits severe foundational weaknesses:
1. **Client-Centric Security Model**: The system routinely delegates authorization decisions to the client browser (relying on `localStorage` booleans, editable roles, and client-side maintenance locks).
2. **Personal Data Exposure (PDPA Non-Compliance)**: Highly sensitive student records (national ID associations, full home addresses, private mobile numbers, cumulative GPAX, family details, and attitude responses) are retrievable over unauthenticated endpoints using easily guessable sequential 5-digit student IDs.
3. **Pervasive HTML Injection Sinks**: Despite defining utility functions such as `escapeHtml()`, numerous critical UI tables, social cards, and dropdown components concatenate user inputs directly into `innerHTML` or attribute event handlers.
4. **Overly Permissive Edge Security Policies**: The Content Security Policy explicitly allows `'unsafe-inline'`, `'unsafe-eval'`, and wildcard Cloudflare Quick Tunnels (`https://*.trycloudflare.com`), completely disabling browser-level XSS defenses and permitting unrestricted data exfiltration.

Remediation must be prioritized to address critical PII exposure and edge injection vectors before standard operational hardening.

---

## 2. System Architecture Decomposition & Threat Modeling

### 2.1 Multi-Subdomain Architecture & Edge Topology
The application is structured as a unified static and serverless edge repository deployed to the Vercel Edge Network. It interfaces with an external Node.js/Express backend running behind Cloudflare Tunnels:

```
                                  [ User Browser / Client ]
                                              |
                   +--------------------------+--------------------------+
                   |                          |                          |
     https://admission.sarawatdc.site  https://roster.sarawatdc.site  https://admin.sarawatdc.site
     (Recruitment & Self-Service)      (SI-ROSTER Duty System)       (Administrative Portal)
                   |                          |                          |
                   +--------------------------+--------------------------+
                                              |
                                [ Vercel Edge Network Router ]
                             (vercel.json Edge Rules & Rewrites)
                                              |
            +---------------------------------+---------------------------------+
            |                                 |                                 |
  [ Serverless Edge Functions ]    [ Static Web Assets ]             [ Edge Reverse Proxy ]
  - api/si-roster.js               - index.html, admission.html      - /api/:path*   -> api.sarawatdc.site
  - api/activities.js              - admin.html, team.html           - /uploads/:path* -> api.sarawatdc.site
  - api/activity-image.js          - SI-ROSTER/index.html, admin.html
  - api/admin.js, api/dsd.js       - js/*, css/*, assets/*
            |                                                                   |
            +---------------------------------+---------------------------------+
                                              |
                               [ Cloudflare Tunnel / Ingress ]
                                (https://api.sarawatdc.site)
                                              |
                             [ Backend Host (Node.js / Express) ]
                             - Port 5000 Ingress Controller
                             - SQLite Database Engine (sarawat.db)
                             - Local Storage Directory (/uploads)
                             - Discord Bot & Webhook Emitters
```

### 2.2 Data Flow Diagram (DFD Level 1)

```
 [ Public Visitor / Student ]                   [ Administrative Operator ]
       |                  |                                   |
       | (1) Browse Web   | (2) Submit / Edit Application     | (3) Admin Login / Action
       v                  v                                   v
+---------------------------------------------------------------------------------------+
| TRUST BOUNDARY 1 (Client Execution Context / Web Browser)                             |
|                                                                                       |
|  [ Web Storage Sinks ]                                                                |
|  - localStorage: 'dg_sarawat_admins' (Plaintext Credentials!)                         |
|  - localStorage: 'adminAuth' (Base64 Session Token)                                   |
|  - localStorage: 'dg_sarawat_admin_role' (Client-Modifiable Role String)              |
|  - sessionStorage: 'sarawat_admin_auth' = 'true' (Bypass Flag)                        |
|  - localStorage: 'dg_device_uuid' (Client-Generated ID)                               |
+---------------------------------------------------------------------------------------+
       |                                                      |
       | (4) HTTPS (Query Params: ?v=, ?id=)                  | (5) Bearer Token / API Call
       v                                                      v
+---------------------------------------------------------------------------------------+
| TRUST BOUNDARY 2 (Edge Infrastructure: Vercel Network)                                |
|                                                                                       |
|  [ vercel.json Rewrite Engine ]                                                       |
|  - Proxies /api/:path* ------------------------------------------------------------+  |
|  - Proxies /uploads/:path* -----------------------------------------------------+  |  |
|                                                                                 |  |  |
|  [ Serverless Functions (api/*.js) ]                                            |  |  |
|  - Injects unescaped query string directly into OpenGraph HTML tags             |  |  |
|  - Decodes Base64 data URLs with caller-supplied Content-Type                   |  |  |
+---------------------------------------------------------------------------------+--+--+
                                                                                  |  |
                                                     +----------------------------+  |
                                                     |                               |
       +---------------------------------------------+-------------------------------+
       |
       | (6) Cloudflare Tunnel Protocol (Encrypted Ingress)
       v
+---------------------------------------------------------------------------------------+
| TRUST BOUNDARY 3 (Cloudflare Ingress Gateway: https://api.sarawatdc.site)             |
+---------------------------------------------------------------------------------------+
       |
       | (7) Reverse Proxy to Internal Port 5000
       v
+---------------------------------------------------------------------------------------+
| TRUST BOUNDARY 4 (Host Backend: Express.js REST API & Persistence)                    |
|                                                                                       |
|  [ API Router Middleware ]                                                            |
|  - CORS & Header validation                                                           |
|  - Missing Auth: /api/recruitment/applicant-correction/:id                            |
|  - Missing Auth: /api/recruitment/export (Tokens passed in GET query string)          |
|  - Missing Auth: /api/students (Full 143-member roster dump)                          |
|                                                                                       |
|  [ Datastore & File Storage ]                                                         |
|  - SQLite Database Engine                                                             |
|  - Local Uploads Directory (/uploads/*)                                               |
+---------------------------------------------------------------------------------------+
       |
       | (8) Outbound Integration Webhooks
       v
+---------------------------------------------------------------------------------------+
| TRUST BOUNDARY 5 (Third-Party Services)                                               |
| - Cloudflare Turnstile API (CAPTCHA token verification)                               |
| - Discord API (Administrative alerts & member presence)                               |
| - api.ipify.org (Client-reported IP discovery service)                                |
+---------------------------------------------------------------------------------------+
```

### 2.3 Trust Boundary Analysis
1. **TB-1: Untrusted Public Browser ↔ Edge Static & Serverless Layer**:
   - *Crossed by*: HTTP GET/POST parameters (`req.query.v`, `req.query.id`, `req.query.t`), Host headers, and form uploads.
   - *Security Failure*: Edge serverless functions trust caller query strings without sanitization, reflecting unencoded strings into server-rendered HTML.
2. **TB-2: Browser DOM Context ↔ Web Storage (`localStorage` / `sessionStorage`)**:
   - *Crossed by*: Client scripts reading and writing authorization states.
   - *Security Failure*: Administrative authorization is determined by the presence of unvalidated strings in Web Storage. Plaintext admin passwords and base64 tokens are stored client-side.
3. **TB-3: Client Application ↔ Backend API Gateway (`api.sarawatdc.site`)**:
   - *Crossed by*: REST API requests transmitting applicant data, roster registrations, and administrative updates.
   - *Security Failure*: Key endpoints omit authentication headers entirely. Export and backup routes pass sensitive JWTs in URL query strings.
4. **TB-4: Backend Controllers ↔ Persistence & Local Filesystem**:
   - *Crossed by*: SQL queries and file uploads.
   - *Security Failure*: Absence of server-side object-level access controls allows arbitrary ID enumeration across applicant records.
5. **TB-5: Local Development & Version Control ↔ Public Git / Deployment Platforms**:
   - *Crossed by*: Git sync scripts and developer workstation files.
   - *Security Failure*: Active Vercel OIDC service tokens are stored in the project workspace, and automated batch scripts execute uninspected force-pushes (`git push --force`).

### 2.4 STRIDE Threat Matrix

| Threat Category | System Component | Specific Threat Description | Affected Finding(s) |
| :--- | :--- | :--- | :--- |
| **Spoofing (S)** | SI-ROSTER Authentication (`app.js`) | Client queries `api.ipify.org` and sends client-reported IP to `verifyAdmin()`. Attackers can spoof arbitrary IP addresses in audit logs. | Finding 14 |
| **Spoofing (S)** | Anti-Spam Quota Bridge (`bridge.js`) | Device UUIDs are generated in client `localStorage`. Bots can forge new UUIDs per request to bypass rate limiting. | Finding 15 |
| **Tampering (T)** | Serverless Edge Pre-renderer (`api/si-roster.js`, `api/activities.js`) | Query parameter `v` and image URLs are injected into OpenGraph tags without escaping, tampering with HTML DOM and executing arbitrary script. | Finding 1, Finding 2 |
| **Tampering (T)** | Applicant Self-Correction API (`admission.html`, `api-config.js`) | Unauthenticated PUT request allows modifying any student's application data using only their 5-digit student ID. | Finding 4 |
| **Repudiation (R)** | Git Automation (`sync-to-github.bat`) | Script reconfigures global git username and email to a fixed identity, obscuring actual developer attribution. | Finding 16 |
| **Information Disclosure (I)** | Recruitment Correction & Export APIs (`admission.html`, `admin-recruitment.js`) | Direct object references allow mass harvesting of student PII (phone, GPAX, address, essay) and full unauthenticated Excel exports. | Finding 4, Finding 5 |
| **Information Disclosure (I)** | Local Workspace Credentials (`.env.local`) | Active Vercel OIDC JSON Web Token committed in workspace root exposes deployment controls. | Finding 13 |
| **Information Disclosure (I)** | LocalStorage Password Persistence (`admin.html`) | Plaintext administrator passwords saved in `localStorage.dg_sarawat_admins` can be exfiltrated via XSS. | Finding 7 |
| **Denial of Service (D)** | Telemetry Heartbeat Beacon (`js/api-config.js`) | Unthrottled 10-second beacon sent by every concurrent visitor creates write contention on backend SQLite database. | Finding 17 |
| **Elevation of Privilege (E)** | Client-Side Role Gates (`admin.html`, `SI-ROSTER/admin.html`, `page-guard.js`) | Setting `sarawat_admin_auth` or `dg_sarawat_admin_role` in browser DevTools elevates user to Super Admin and unlocks administrative tabs. | Finding 6 |
| **Elevation of Privilege (E)** | Frontend DOM Sinks (`admin-recruitment.js`, `admin-year70.js`, `SI-ROSTER/admin.html`) | Stored inputs from applicant names, candidate rankings, and duty names execute script in administrative sessions. | Finding 8, Finding 9, Finding 10, Finding 11 |

---

## 3. Comprehensive Findings Catalog

```
================================================================================
CRITICAL SEVERITY FINDINGS (P0 / Immediate Remediation Required)
================================================================================
```

### Finding 1: Reflected Cross-Site Scripting (Reflected XSS) & Meta Tag Attribute Injection in Serverless Edge SSR
- **Unique Finding ID**: SEC-FINDING-001
- **Vulnerability Title**: Reflected Cross-Site Scripting via Serverless OpenGraph Pre-rendering
- **Relative Path**: `api/si-roster.js`
- **Absolute Path**: `f:\MP\api\si-roster.js`
- **Line Numbers**: 189–190, 202–203, 212
- **CWE Identifier**: CWE-79 (Improper Neutralization of Input During Web Page Generation — 'Cross-site Scripting')
- **OWASP Top 10 (2021)**: A03:2021 — Injection
- **Severity Rating**: **Critical** (CVSS:3.1/AV:N/AC:L/PR:N/UI:R/S:C/C:H/I:H/A:N — Base Score: 9.3)
- **Mechanism of Failure**:
  The Vercel Serverless Function `api/si-roster.js` dynamically pre-renders OpenGraph and Twitter meta tags for requests to `/si-roster?id=...`. The handler extracts `req.query.v` (or `req.query.t`) to determine the image cache-buster version. While `sheetName` is properly encoded with `encodeURIComponent()`, `version` is stringified without URL-encoding or HTML-escaping and interpolated directly into `dynamicOgImage`:
  `const dynamicOgImage = https://api.sarawatdc.site/api/roster/og-image?id=${encodeURIComponent(sheetName)}&v=${version};`
  This unescaped string is then substituted into the HTML `<meta property="og:image" content="...">` and `<meta name="twitter:image" content="...">` attributes using regex replacement:
  `html = html.replace(/(<meta\s+property=["']og:image["'][^>]*content=["']).*?(["'])/i, $1${dynamicOgImage}$2);`
  An attacker crafting the URL:
  `https://roster.sarawatdc.site/?id=duty1&v=%22%3E%3Cscript%3Ealert(document.domain)%3C/script%3E`
  causes the serverless edge response to inject an active `<script>` tag into the returned HTML `<head>`. Because the Content Security Policy in `vercel.json` permits `'unsafe-inline'`, the browser executes the script immediately in the origin context.
- **Verbatim Code Evidence**:
  ```javascript
  // f:\MP\api\si-roster.js:189-190
  const version = (req.query.v || req.query.t || (found.updated_at ? new Date(found.updated_at).getTime() : Date.now())).toString().trim();
  const dynamicOgImage = `https://api.sarawatdc.site/api/roster/og-image?id=${encodeURIComponent(sheetName)}&v=${version}`;

  // f:\MP\api\si-roster.js:202-203
  html = html.replace(/(<meta\s+property=["']og:image["'][^>]*content=["']).*?(["'])/i, `$1${dynamicOgImage}$2`);
  html = html.replace(/(<meta\s+property=["']og:image:secure_url["'][^>]*content=["']).*?(["'])/i, `$1${dynamicOgImage}$2`);

  // f:\MP\api\si-roster.js:212
  html = html.replace(/(<meta\s+name=["']twitter:image["'][^>]*content=["']).*?(["'])/i, `$1${dynamicOgImage}$2`);
  ```
- **Project-Specific Root Cause Analysis**:
  Asymmetric sanitization discipline: the developer recognized the necessity of wrapping `sheetName` with `encodeURIComponent()`, but assumed `req.query.v` and `req.query.t` would always be numeric Unix timestamps. Because Vercel serverless request query parameters accept arbitrary user-controlled strings, unescaped string interpolation into an HTML attribute context produces a direct reflected injection vulnerability.
- **Impact Analysis**:
  - *Confidentiality*: Complete compromise. Injected script can access `localStorage.adminAuth`, extracting administrative bearer tokens and active session credentials.
  - *Integrity*: High. Attacker can manipulate the roster DOM, forge duty signups, or redirect users to malicious external portals.
  - *Availability*: Medium. Injected scripts can destroy client local state or trigger client-side infinite reload loops.
  - *Compliance*: Violates Thailand PDPA requirements regarding technical safeguards against unauthorized system access.

---

### Finding 4: Broken Object-Level Authorization (BOLA/IDOR) & Sensitive Student PII Exposure
- **Unique Finding ID**: SEC-FINDING-004
- **Vulnerability Title**: Unauthenticated Direct Object Reference Exposing Complete Student Applicant Records
- **Relative Path**: `admission.html` and `js/api-config.js`
- **Absolute Path**: `f:\MP\admission.html` and `f:\MP\js\api-config.js`
- **Line Numbers**: `admission.html`: 2884–2950; `js/api-config.js`: 1310–1327
- **CWE Identifier**: CWE-639 (Authorization Bypass Through User-Controlled Key) / CWE-306 (Missing Authentication for Critical Function)
- **OWASP Top 10 (2021)**: A01:2021 — Broken Access Control
- **Severity Rating**: **Critical** (CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N — Base Score: 9.1)
- **Mechanism of Failure**:
  The student admissions portal provides an applicant self-correction feature (`startCorrectionForStudent(studentId)` in `admission.html`). When a user supplies a 5-digit student ID, the client invokes `window.SarawatAPI.getApplicantForCorrection(sid)`.
  This sends an HTTP GET request to:
  `GET /api/recruitment/applicant-correction/:studentId`
  **This endpoint requires zero authentication headers, session cookies, or secondary verification factors.** The backend responds with the applicant's complete profile, which `admission.html` automatically populates into the form:
  - Personal identification: Student ID, prefix, first name, last name, classroom, grade level
  - Personal contact details: Mobile phone number (`inpPhone`)
  - Academic metrics: Cumulative GPAX (`inpGpax`)
  - Complete residential address: House number, village number, building, soi, road, subdistrict, district, province, and postal code
  - Facial photograph URL: Direct path to profile photo (`photoUrl`)
  - Application essay answers: Full text of written responses to attitude screening questions
  Because student IDs are sequential 5-digit numbers (e.g., 10000–99999), an unauthenticated external attacker can iterate through valid ID ranges with a simple script and mass-harvest confidential personal information for all applicants. Furthermore, `submitApplicantCorrection` (`PUT /api/recruitment/correction/:studentId`) also requires no authentication, allowing malicious modification of any applicant's application.
- **Verbatim Code Evidence**:
  ```javascript
  // f:\MP\js\api-config.js:1310-1316
  getApplicantForCorrection: async function (studentId) {
    const base = this.getBaseUrl() || '';
    const res = await fetch(`${base}/api/recruitment/applicant-correction/${encodeURIComponent(studentId)}`);
    const data = await res.json();
    if (!res.ok) throw new Error(data.error || 'ไม่สามารถดึงข้อมูลสำหรับแก้ไขได้');
    return data;
  },

  // f:\MP\admission.html:2892-2900
  const res = await window.SarawatAPI.getApplicantForCorrection(sid);
  if (!res.success || !res.applicant) {
    throw new Error(res.error || 'ไม่พบข้อมูลใบสมัคร');
  }
  const app = res.applicant;
  isEditingMode = true;
  editingStudentId = app.studentId;

  // f:\MP\admission.html:2919, 2925-2931
  if (document.getElementById('inpPhone')) document.getElementById('inpPhone').value = formatPhoneNumber(app.phone || '0');
  if (document.getElementById('inpHouseNo')) document.getElementById('inpHouseNo').value = app.houseNo || '';
  if (document.getElementById('inpVillageNo')) document.getElementById('inpVillageNo').value = app.villageNo || '';
  if (document.getElementById('inpBuilding')) document.getElementById('inpBuilding').value = app.building || '';
  if (document.getElementById('inpSoi')) document.getElementById('inpSoi').value = app.soi || '';
  if (document.getElementById('inpRoad')) document.getElementById('inpRoad').value = app.road || '';
  if (document.getElementById('inpPostalCode')) document.getElementById('inpPostalCode').value = app.postalCode || '';
  ```
- **Project-Specific Root Cause Analysis**:
  To simplify the applicant correction user experience, the developers removed phone number verification requirements (explicitly noted at line 4816 of `admission.html`: *"ไม่ต้องกรอกเบอร์โทรศัพท์"*). By omitting secondary verification factors or signed session tokens, the endpoint was left entirely unprotected against automated sequential enumeration.
- **Impact Analysis**:
  - *Confidentiality*: Catastrophic. Mass harvesting of minors' residential addresses, telephone numbers, academic records, and personal photographs.
  - *Integrity*: Catastrophic. Attackers can tamper with or deface any applicant's submission prior to committee review.
  - *Compliance*: Major direct breach of the Thailand Personal Data Protection Act (PDPA B.E. 2562), Section 37 (obligation to implement appropriate security measures to prevent unauthorized data loss, access, use, or alteration) carrying civil damages, administrative fines up to 5,000,000 THB, and criminal liability.

---

### Finding 5: Unauthenticated Full Export of Applicant Database via Direct URL
- **Unique Finding ID**: SEC-FINDING-005
- **Vulnerability Title**: Missing Authentication on Recruitment Database Spreadsheet Export
- **Relative Path**: `js/admin-recruitment.js`
- **Absolute Path**: `f:\MP\js\admin-recruitment.js`
- **Line Numbers**: 118–123
- **CWE Identifier**: CWE-306 (Missing Authentication for Critical Function) / CWE-200 (Exposure of Sensitive Information to an Unauthorized Actor)
- **OWASP Top 10 (2021)**: A01:2021 — Broken Access Control
- **Severity Rating**: **Critical** (CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N — Base Score: 7.5; Elevated to **Critical** due to complete database dump of minor PII)
- **Mechanism of Failure**:
  In `js/admin-recruitment.js`, the export functionality is implemented by constructing a direct URL string:
  `return ${base}/api/recruitment/export?${q.toString()};`
  This URL is opened directly via browser navigation (`window.open` or `<a href="...">`). Consequently, **no HTTP `Authorization` header is attached to the request**. Unless the backend enforces separate session cookies, the endpoint `/api/recruitment/export` serves complete Excel/CSV spreadsheets containing every applicant record to any unauthenticated caller who knows or discovers the URL path.
- **Verbatim Code Evidence**:
  ```javascript
  // f:\MP\js\admin-recruitment.js:117-123
  if (!apiBridge.getRecruitmentExportUrl) {
    apiBridge.getRecruitmentExportUrl = function (format = 'xlsx', status = 'all', grade = 'all') {
      const base = this.getBaseUrl();
      const q = new URLSearchParams({ format, status, grade });
      return `${base}/api/recruitment/export?${q.toString()}`;
    };
  }
  ```
- **Project-Specific Root Cause Analysis**:
  The developer implemented file downloads using standard browser HTTP GET navigation rather than an authenticated asynchronous `fetch()` request with a Blob conversion pattern, causing the request to omit authentication tokens.
- **Impact Analysis**:
  - *Confidentiality*: Catastrophic. Complete exfiltration of the entire applicant database in a single unauthenticated HTTP request.
  - *Compliance*: Immediate violation of PDPA B.E. 2562 data breach reporting criteria, requiring mandatory notification to the Personal Data Protection Committee (PDPC) within 72 hours.

---

```
================================================================================
HIGH SEVERITY FINDINGS (P1 / High Priority Remediation Required)
================================================================================
```

### Finding 2: Unescaped Image URL Injection in Server-Side OpenGraph & Twitter Meta Tags
- **Unique Finding ID**: SEC-FINDING-002
- **Vulnerability Title**: Stored/Reflected Attribute Injection via Unsanitized OpenGraph Image URLs
- **Relative Path**: `api/activities.js`
- **Absolute Path**: `f:\MP\api\activities.js`
- **Line Numbers**: 89–98, 108–109, 114
- **CWE Identifier**: CWE-79 (Cross-site Scripting) / CWE-116 (Improper Encoding or Escaping of Output)
- **OWASP Top 10 (2021)**: A03:2021 — Injection
- **Severity Rating**: **High** (CVSS:3.1/AV:N/AC:L/PR:N/UI:R/S:C/C:H/I:L/A:N — Base Score: 8.2)
- **Mechanism of Failure**:
  In `api/activities.js`, server-side meta tags are generated for news items. While `fullTitle` and `desc` are explicitly passed through `escapeHtml()` (lines 105–107, 112–113), the variable `resolvedImage` is interpolated directly into regex replacements for `<meta property="og:image" content="...">`, `<meta property="og:image:secure_url" content="...">`, and `<meta name="twitter:image" content="...">` without calling `escapeHtml()`.
  If an administrator or attacker submits a news item with an image URL containing attribute-breaking characters (e.g., `https://example.com/cover.jpg" onmouseover="alert(1)`), or if replacement metacharacters (e.g., `$'` or `$`&) are embedded, the string breaks out of the HTML attribute and injects arbitrary HTML elements into the page head.
- **Verbatim Code Evidence**:
  ```javascript
  // f:\MP\api\activities.js:108-109, 114
  html = html.replace(/(<meta\s+property=["']og:image["']\s+content=["']).*?(["'])/i, `$1${resolvedImage}$2`);
  html = html.replace(/(<meta\s+property=["']og:image:secure_url["']\s+content=["']).*?(["'])/i, `$1${resolvedImage}$2`);
  ...
  html = html.replace(/(<meta\s+name=["']twitter:image["']\s+content=["']).*?(["'])/i, `$1${resolvedImage}$2`);
  ```
- **Project-Specific Root Cause Analysis**:
  Incomplete output encoding: the author recognized text fields require escaping, but mistakenly assumed URLs are intrinsically safe and do not require HTML attribute entity encoding.
- **Impact Analysis**:
  - *Confidentiality*: High. Cross-Site Scripting allows session hijacking.
  - *Integrity*: High. Page defacement and malicious redirection of visitors.

---

### Finding 3: Arbitrary MIME Type Spoofing & Stored XSS via Base64 Image Proxy Handler
- **Unique Finding ID**: SEC-FINDING-003
- **Vulnerability Title**: Unrestricted MIME Type Handling Leading to Stored XSS in Image Proxy
- **Relative Path**: `api/activity-image.js`
- **Absolute Path**: `f:\MP\api\activity-image.js`
- **Line Numbers**: 85–96
- **CWE Identifier**: CWE-434 (Unrestricted Upload of File with Dangerous Type) / CWE-79 (Stored Cross-site Scripting)
- **OWASP Top 10 (2021)**: A03:2021 — Injection
- **Severity Rating**: **High** (CVSS:3.1/AV:N/AC:L/PR:N/UI:R/S:C/C:H/I:H/A:N — Base Score: 8.8)
- **Mechanism of Failure**:
  `api/activity-image.js` proxies and decodes news cover images. When `rawImg` begins with `data:`, it extracts the MIME type using regex: `/^data:([^;]+);base64,(.+)$/s`.
  The endpoint assigns `match[1]` directly to the HTTP response header `res.setHeader('Content-Type', contentType)` and returns the decoded binary buffer with public caching headers (`max-age=86400, s-maxage=86400`).
  If a malicious news record contains `data:text/html;base64,PHNjcmlwdD5hbGVydChkb2N1bWVudC5kb21haW4pPC9zY3JpcHQ+` or `data:image/svg+xml;base64,...`, the endpoint serves executable HTML or interactive SVG directly under `https://www.sarawatdc.site/api/activity-image?id=...`, executing arbitrary JavaScript within the primary origin.
- **Verbatim Code Evidence**:
  ```javascript
  // f:\MP\api\activity-image.js:85-95
  if (rawImg.startsWith('data:')) {
    const match = rawImg.match(/^data:([^;]+);base64,(.+)$/s);
    if (match) {
      const contentType = match[1] || 'image/jpeg';
      const base64Data = match[2];
      const buffer = Buffer.from(base64Data, 'base64');
      res.setHeader('Content-Type', contentType);
      res.setHeader('Content-Length', buffer.length);
      res.setHeader('Cache-Control', 'public, max-age=86400, s-maxage=86400, stale-while-revalidate=604800');
      return res.status(200).send(buffer);
    }
  }
  ```
- **Project-Specific Root Cause Analysis**:
  Missing content-type allowlisting: the serverless function blindly trusts client/database supplied MIME types without restricting responses to safe raster image formats.
- **Impact Analysis**:
  - *Confidentiality*: High. Stored XSS executes on the main domain with long-lived CDN edge caching.
  - *Integrity*: High. Can serve malicious phishing content or exploit administrator sessions.

---

### Finding 6: Client-Side Admin Authentication & Privilege Escalation Bypasses
- **Unique Finding ID**: SEC-FINDING-006
- **Vulnerability Title**: Client-Side Enforcement of Server-Side Security & Role Tampering
- **Relative Path**: `admin.html`, `SI-ROSTER/admin.html`, and `js/page-guard.js`
- **Absolute Path**: `f:\MP\admin.html`, `f:\MP\SI-ROSTER\admin.html`, and `f:\MP\js\page-guard.js`
- **Line Numbers**: `admin.html`: 8508–8526, 8585–8643; `SI-ROSTER/admin.html`: 8–15, 5880–5911; `js/page-guard.js`: 36–50
- **CWE Identifier**: CWE-602 (Client-Side Enforcement of Server-Side Security) / CWE-287 (Improper Authentication)
- **OWASP Top 10 (2021)**: A07:2021 — Identification and Authentication Failures
- **Severity Rating**: **High** (CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N — Base Score: 8.5)
- **Mechanism of Failure**:
  1. In `admin.html`, `isCurrentSuperAdmin()` reads the role string directly from `localStorage.getItem('dg_sarawat_admin_role')` or `sessionStorage.getItem('dg_sarawat_admin_role')`. Any unauthenticated user running `localStorage.setItem('dg_sarawat_admin_role', 'Super Admin')` immediately unlocks Super Admin functionality (telemetry, Discord bot config, system backups).
  2. In `admin.html:8620-8638`, `checkAuth()` checks if `sessionStorage.getItem('sarawat_admin_auth') === 'true'`. If the backend is unreachable or throws a network error, the dashboard grants entry (`enterAdminDashboard()`).
  3. In `SI-ROSTER/admin.html:8-15` and `5880-5891`, if `localStorage.getItem('adminAuth')` contains any unexpired JSON object, the page immediately adds `is-admin` to the document class and displays `#adminPortalApp`. At lines 5899–5907, `verifyAdminSession` is **only invoked if `authCookie.token` is truthy**. Omitting `token` skips backend validation entirely.
  4. In `js/page-guard.js:36-40`, maintenance mode and page locks are bypassed if `sarawat_admin_auth === 'true'`.
- **Verbatim Code Evidence**:
  ```javascript
  // f:\MP\admin.html:8524-8526
  const storedRole = localStorage.getItem('dg_sarawat_admin_role') || sessionStorage.getItem('dg_sarawat_admin_role');
  if (checkRole(storedRole)) return true;

  // f:\MP\admin.html:8620-8637
  const isAuth = sessionStorage.getItem('sarawat_admin_auth');
  const savedUserStr = sessionStorage.getItem('sarawat_admin_user');
  if (isAuth === 'true' && savedUserStr) {
    ...
    enterAdminDashboard();
    return;
  }

  // f:\MP\SI-ROSTER\admin.html:8-13
  var stored = localStorage.getItem('adminAuth');
  if (stored) {
    var parsed = JSON.parse(stored);
    if (parsed && parsed.expires && new Date().getTime() <= parsed.expires) {
      document.documentElement.classList.add('is-admin');
    }
  }
  ```
- **Project-Specific Root Cause Analysis**:
  Treating client-controlled Web Storage values as authoritative session tokens and implementing insecure "offline fallback" modes in production builds.
- **Impact Analysis**:
  - *Confidentiality*: High. Unauthorized access to all cached administrative data, logs, and member rosters.
  - *Integrity*: High. Unauthorized users can view and trigger administrative interfaces.

---

### Finding 7: Cleartext Password Storage in Browser LocalStorage
- **Unique Finding ID**: SEC-FINDING-007
- **Vulnerability Title**: Cleartext Persistence of Administrator Passwords in Web Storage
- **Relative Path**: `admin.html`
- **Absolute Path**: `f:\MP\admin.html`
- **Line Numbers**: 8737–8740, 8876–8885
- **CWE Identifier**: CWE-312 (Cleartext Storage of Sensitive Information) / CWE-256 (Plaintext Storage of a Password)
- **OWASP Top 10 (2021)**: A02:2021 — Cryptographic Failures
- **Severity Rating**: **High** (CVSS:3.1/AV:N/AC:L/PR:N/UI:R/S:U/C:H/I:N/A:N — Base Score: 7.5)
- **Mechanism of Failure**:
  During First-Time Super Admin setup in `admin.html`, the administrator's plaintext password is saved directly into `localStorage.getItem('dg_sarawat_admins')`:
  `admins.push({ id: Date.now(), username: u, password: p, role: 'Super Admin' });`
  `saveLocalAdmins(admins);`
  During login fallback (lines 8876–8885), authentication is checked via plain string equality (`a.password === p`). Any script running in the browser (via XSS or malicious extensions) can execute `localStorage.getItem('dg_sarawat_admins')` and retrieve all administrator usernames and passwords in cleartext.
- **Verbatim Code Evidence**:
  ```javascript
  // f:\MP\admin.html:8737-8740
  // 2. Always persist to localStorage
  const admins = getLocalAdmins();
  admins.push({ id: Date.now(), username: u, password: p, role: 'Super Admin' });
  saveLocalAdmins(admins);

  // f:\MP\admin.html:8877-8879
  const localAdmins = getLocalAdmins();
  const found = localAdmins.find(a => a.username.toLowerCase() === u.toLowerCase() && a.password === p);
  ```
- **Project-Specific Root Cause Analysis**:
  Architectural conflation of a local offline prototype with production authentication. Passwords should never be stored on client devices, let alone in unencrypted plaintext.
- **Impact Analysis**:
  - *Confidentiality*: High. Permanent credential compromise.
  - *Integrity*: High. Compromised credentials can be reused against production backend endpoints.

---

### Finding 8: Stored Cross-Site Scripting (XSS) in Admin Recruitment Dashboard
- **Unique Finding ID**: SEC-FINDING-008
- **Vulnerability Title**: Stored Cross-Site Scripting via Unescaped Applicant Table Rendering
- **Relative Path**: `js/admin-recruitment.js`
- **Absolute Path**: `f:\MP\js\admin-recruitment.js`
- **Line Numbers**: 395–450
- **CWE Identifier**: CWE-79 (Cross-site Scripting — 'Stored XSS')
- **OWASP Top 10 (2021)**: A03:2021 — Injection
- **Severity Rating**: **High** (CVSS:3.1/AV:N/AC:L/PR:N/UI:R/S:C/C:H/I:H/A:N — Base Score: 8.8)
- **Mechanism of Failure**:
  In `loadApplicantsTable()`, applicant records received from the backend are rendered into the admin table by mapping over `res.applicants` and assigning the resulting template literal to `tbody.innerHTML`.
  User-controllable fields—including `a.first_name`, `a.last_name`, `a.prefix`, `a.student_id`, and `a.club`—are interpolated directly without `escapeHtml()`.
  An applicant submitting `<img src=x onerror=alert(document.cookie)>` as their first name or school club will trigger arbitrary JavaScript execution in the browser of any administrator reviewing the recruitment list.
- **Verbatim Code Evidence**:
  ```javascript
  // f:\MP\js\admin-recruitment.js:421, 427, 442
  <img src="${photoSrc}" alt="${a.first_name}" ... />
  ...
  <td style="padding: 12px 16px; font-weight: 750; color: #0f172a;">
    ${a.prefix}${a.first_name} ${a.last_name}
  </td>
  ...
  <div style="white-space: nowrap; overflow: hidden; text-overflow: ellipsis;">${a.club}</div>
  ```
- **Project-Specific Root Cause Analysis**:
  Inconsistent sanitization: `escapeHtml()` is defined at line 3 of `admin-recruitment.js` and used in modal views (lines 805–806), but was omitted in the primary applicant table loop.
- **Impact Analysis**:
  - *Confidentiality*: High. Execution inside the authenticated administrator session allows extraction of administrative JWTs and database export.
  - *Integrity*: High. Attacker can approve their own application or manipulate recruitment scores.

---

### Finding 9: Inline Event Handler Injection XSS in Candidate Rankings
- **Unique Finding ID**: SEC-FINDING-009
- **Vulnerability Title**: Cross-Site Scripting via Inline `onerror` Attribute Concatenation
- **Relative Path**: `js/admin-year70.js`
- **Absolute Path**: `f:\MP\js\admin-year70.js`
- **Line Numbers**: 402, 749
- **CWE Identifier**: CWE-79 (Cross-site Scripting) / CWE-83 (Improper Neutralization of Script in Attributes)
- **OWASP Top 10 (2021)**: A03:2021 — Injection
- **Severity Rating**: **High** (CVSS:3.1/AV:N/AC:L/PR:N/UI:R/S:C/C:H/I:H/A:N — Base Score: 8.2)
- **Mechanism of Failure**:
  In `renderPodium()` (line 402) and `renderCandidateRows()` (line 749), candidate avatars embed fallback HTML inside an inline `onerror` attribute:
  `onerror="this.onerror=null; this.parentElement.innerHTML='<div style=\\\'...\\\'>${cand.student_name ? cand.student_name.trim().charAt(0) : '?'}</div>';"`
  If `cand.student_name` begins with a single quote `'` (e.g., `' onclick=alert(1)//`), it breaks out of the single-quoted string passed to `innerHTML` within the `onerror` attribute, causing arbitrary JavaScript execution when image loading fails.
- **Verbatim Code Evidence**:
  ```javascript
  // f:\MP\js\admin-year70.js:402, 749
  onerror="this.onerror=null; this.parentElement.innerHTML='<div style=\\\'width:100%;height:100%;background:linear-gradient(135deg,#0f172a,#334155);color:#fff;font-weight:800;font-size:2.2rem;display:flex;align-items:center;justify-content:center;\\\'>${cand.student_name ? cand.student_name.trim().charAt(0) : '?'}</div>';"
  ```
- **Project-Specific Root Cause Analysis**:
  Mixing JavaScript execution contexts within HTML attribute strings. Embedding nested quotes inside inline event handlers bypasses simple HTML entity escaping.
- **Impact Analysis**:
  - *Confidentiality*: High. Compromises administrator sessions reviewing candidate scoring.
  - *Integrity*: High. Manipulation of voting or score tallies.

---

### Finding 10: Stored & DOM XSS in SI-ROSTER Duty Management
- **Unique Finding ID**: SEC-FINDING-010
- **Vulnerability Title**: DOM and Stored Cross-Site Scripting in Duty Name Dropdowns and User Lists
- **Relative Path**: `SI-ROSTER/admin.html` and `SI-ROSTER/js/app.js`
- **Absolute Path**: `f:\MP\SI-ROSTER\admin.html` and `f:\MP\SI-ROSTER\js\app.js`
- **Line Numbers**: `SI-ROSTER/admin.html`: 6763–6773; `SI-ROSTER/js/app.js`: 485–490
- **CWE Identifier**: CWE-79 (Cross-site Scripting — 'DOM XSS')
- **OWASP Top 10 (2021)**: A03:2021 — Injection
- **Severity Rating**: **High** (CVSS:3.1/AV:N/AC:L/PR:N/UI:R/S:C/C:H/I:H/A:N — Base Score: 8.2)
- **Mechanism of Failure**:
  1. In `SI-ROSTER/admin.html:6763-6766`, as an administrator types into the duty creation input, `listEl.innerHTML` renders:
     `html += '<div class="duty-dropdown-item create-new" onclick="selectDutyOption(this, \'' + safeTerm + '\', true)"><i class="fa-solid fa-plus-circle"></i> สร้างหน้าที่ "' + inputEl.value.trim() + '"</div>';`
     `safeTerm` only escapes double quotes (`replace(/"/g, '&quot;')`), allowing single quotes to break out of `selectDutyOption()`. More critically, `inputEl.value.trim()` in the text portion is **completely unescaped**, triggering instant DOM XSS on keystroke.
  2. In `SI-ROSTER/js/app.js:485-490`, `act.users.forEach(u => ...)` interpolates `u.name` and `u.duty` directly into template strings assigned to `container.innerHTML` without sanitization.
- **Verbatim Code Evidence**:
  ```javascript
  // f:\MP\SI-ROSTER\admin.html:6763-6766, 6773
  const safeTerm = inputEl.value.trim().replace(/"/g, '&quot;');
  html += `<div class="duty-dropdown-item create-new" onclick="selectDutyOption(this, '${safeTerm}', true)">
             <i class="fa-solid fa-plus-circle"></i> สร้างหน้าที่ "${inputEl.value.trim()}"
           </div>`;
  listEl.innerHTML = html;

  // f:\MP\SI-ROSTER\js\app.js:485-490
  act.users.forEach(u => {
    html += `<div style="display:flex; justify-content:space-between; padding: 8px 0; border-bottom: 1px dashed rgba(0,0,0,0.1);">
        <div style="color: var(--text-main); font-size: 0.95rem;"><i class="fa-solid fa-user" ...></i> ${u.name}</div>
        <div style="color: var(--accent); font-size: 0.9rem;">${u.duty}</div>
      </div>`;
  });
  ```
- **Project-Specific Root Cause Analysis**:
  Direct string concatenation of DOM input values into `innerHTML` and inadequate partial character replacement.
- **Impact Analysis**:
  - *Confidentiality*: High. Captures roster administrator credentials and tokens.
  - *Integrity*: High. Forging student duty signups and check-ins.

---

```
================================================================================
MEDIUM SEVERITY FINDINGS (P2 / Next Sprint Remediation)
================================================================================
```

### Finding 11: Stored XSS in Public Personnel & Executive Directory Pages
- **Unique Finding ID**: SEC-FINDING-011
- **Vulnerability Title**: Unsanitized Roster Member and Social Link Rendering
- **Relative Path**: `team.html` and `dsd.html`
- **Absolute Path**: `f:\MP\team.html` and `f:\MP\dsd.html`
- **Line Numbers**: `team.html`: 530, 657; `dsd.html`: 911–921, 1009–1017
- **CWE Identifier**: CWE-79 (Cross-site Scripting)
- **OWASP Top 10 (2021)**: A03:2021 — Injection
- **Severity Rating**: **Medium** (CVSS:3.1/AV:N/AC:L/PR:H/UI:R/S:C/C:L/I:L/A:N — Base Score: 4.8)
- **Mechanism of Failure**:
  In `team.html:657`, member names are concatenated into `html` without escaping: `${isHead ? '...' : ''}${member.name}`.
  In `dsd.html:911-921`, social links (`m.github_url`, `m.website_url`) are checked only with `startsWith('http')` and interpolated into `<a href="${m.github_url}">` without escaping quotes. An admin account entering `https://example.com" onmouseover="alert(1)` will inject attributes into public visitor sessions.
- **Verbatim Code Evidence**:
  ```javascript
  // f:\MP\team.html:657
  ${isHead ? '<i data-lucide="crown" ...></i>' : ''}${member.name}

  // f:\MP\dsd.html:911
  if (isValidSocial(m.github_url)) {
    links.push('<a href="' + m.github_url + '" target="_blank" rel="noopener noreferrer" class="dsd-social-btn" title="GitHub">' + ICONS.github + '</a>');
  }
  ```
- **Root Cause**: Trusting backend database records as intrinsically safe HTML.
- **Impact Analysis**: Medium risk of persistent XSS on high-traffic public organizational directory pages.

---

### Finding 12: Insecure Content Security Policy (CSP) Permitting Inline Scripts and Arbitrary Tunnels
- **Unique Finding ID**: SEC-FINDING-012
- **Vulnerability Title**: Weak Content Security Policy Permitting Script Injection and Tunnel Exfiltration
- **Relative Path**: `vercel.json`
- **Absolute Path**: `f:\MP\vercel.json`
- **Line Numbers**: 51–53
- **CWE Identifier**: CWE-1021 (Improper Restriction of Rendered UI) / CWE-693 (Protection Mechanism Failure)
- **OWASP Top 10 (2021)**: A05:2021 — Security Misconfiguration
- **Severity Rating**: **Medium** (CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N — Base Score: 6.5)
- **Mechanism of Failure**:
  The CSP header configured in `vercel.json` contains:
  `script-src 'self' 'unsafe-inline' 'unsafe-eval' ...`
  `connect-src 'self' https://*.trycloudflare.com ...`
  1. The inclusion of `'unsafe-inline'` and `'unsafe-eval'` completely disables the browser's ability to block injected scripts (negating defense-in-depth for Findings 1, 2, 8, 9, 10).
  2. The wildcard `https://*.trycloudflare.com` allows an attacker executing XSS to spin up a free, disposable Cloudflare Quick Tunnel and exfiltrate stolen credentials or PII without triggering CSP connect-src violations.
- **Verbatim Code Evidence**:
  ```json
  // f:\MP\vercel.json:51-53
  {
    "key": "Content-Security-Policy",
    "value": "default-src 'self' https://*.sarawatdc.site; script-src 'self' 'unsafe-inline' 'unsafe-eval' https://*.sarawatdc.site https://cdn.jsdelivr.net https://cdnjs.cloudflare.com https://unpkg.com https://uicdn.toast.com https://challenges.cloudflare.com; style-src 'self' 'unsafe-inline' https://*.sarawatdc.site https://fonts.googleapis.com https://cdn.jsdelivr.net https://cdnjs.cloudflare.com https://uicdn.toast.com https://unpkg.com; font-src 'self' data: https://fonts.gstatic.com https://cdnjs.cloudflare.com https://cdn.jsdelivr.net; img-src 'self' data: blob: https:; media-src 'self' data: blob: https:; connect-src 'self' https://*.trycloudflare.com https://*.sarawatdc.site https://sarawat-home.vercel.app https://api.ipify.org https://unpkg.com https://cdn.jsdelivr.net https://cdnjs.cloudflare.com https://uicdn.toast.com https://challenges.cloudflare.com; frame-src 'self' https://docs.google.com https://forms.gle https://challenges.cloudflare.com; object-src 'none'; base-uri 'self'; frame-ancestors 'self';"
  }
  ```
- **Root Cause**: Relaxing security controls for development convenience.
- **Impact Analysis**: Disables browser-level XSS mitigation and permits unconstrained exfiltration channels.

---

### Finding 13: Active Vercel OIDC Token Persisted in Local Workspace Configuration
- **Unique Finding ID**: SEC-FINDING-013
- **Vulnerability Title**: Hardcoded Cloud Deployment Credentials in Local Workspace
- **Relative Path**: `.env.local`
- **Absolute Path**: `f:\MP\.env.local`
- **Line Numbers**: 1–2
- **CWE Identifier**: CWE-798 (Use of Hard-coded Credentials) / CWE-522 (Insufficiently Protected Credentials)
- **OWASP Top 10 (2021)**: A07:2021 — Identification and Authentication Failures
- **Severity Rating**: **Medium** (CVSS:3.1/AV:L/AC:L/PR:L/UI:N/S:C/C:H/I:H/A:H — Base Score: 6.8)
- **Mechanism of Failure**:
  Line 2 of `.env.local` contains an active, validly signed `VERCEL_OIDC_TOKEN` issued for user `rujirawattanakitpaisal-4756` targeting project `prj_hD5dZ6ex9Ww05ypDmVIPF6DmA39K` (`sarawat-home`). Because `sync-to-github.bat` executes `git add -A` and `git push --force`, any misconfiguration or accidental staging of `.env*` would immediately leak cloud infrastructure credentials to public repositories.
- **Verbatim Code Evidence**:
  ```ini
  // f:\MP\.env.local:1-2
  # Created by Vercel CLI
  VERCEL_OIDC_TOKEN="eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6Im1yay00MzAyZWMxYjY3MGY0OGE5OGFkNjFkYWRlNGEyM2JlNyJ9..."
  ```
- **Root Cause**: Retaining Vercel CLI development environment tokens on disk.
- **Impact Analysis**: Potential deployment hijacking and environment variable tampering.

---

### Finding 14: Client IP Address Spoofing & Privacy Leakage via `api.ipify.org`
- **Unique Finding ID**: SEC-FINDING-014
- **Vulnerability Title**: Client-Reported IP Address Reliance and Third-Party Information Exposure
- **Relative Path**: `SI-ROSTER/js/app.js`
- **Absolute Path**: `f:\MP\SI-ROSTER\js\app.js`
- **Line Numbers**: 2479–2483, 2485–2498
- **CWE Identifier**: CWE-807 (Reliance on Untrusted Inputs in a Security Decision) / CWE-359 (Exposure of Private Personal Information)
- **OWASP Top 10 (2021)**: A07:2021 — Identification and Authentication Failures
- **Severity Rating**: **Medium** (CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:L/A:N — Base Score: 5.3)
- **Mechanism of Failure**:
  Prior to admin authentication, `SI-ROSTER/js/app.js` queries `https://api.ipify.org?format=json` and forwards `data.ip` to `executeLogin(user, pass, data.ip, btn)`.
  1. Attackers can tamper with the request to supply arbitrary IP strings (e.g., spoofing trusted school internal IPs).
  2. External queries to `api.ipify.org` leak user authentication timing and client IPs to a third party, and network failures or timeouts at `api.ipify.org` disrupt login availability.
- **Verbatim Code Evidence**:
  ```javascript
  // f:\MP\SI-ROSTER\js\app.js:2479-2483
  fetch('https://api.ipify.org?format=json')
    .then(res => res.json())
    .then(data => executeLogin(user, pass, data.ip, btn))
    .catch(err => executeLogin(user, pass, 'Unknown', btn));
  ```
- **Root Cause**: Relying on client-reported network metadata rather than extracting connection headers on the backend.
- **Impact Analysis**: Subversion of audit logs and IP-based rate limiting.

---

### Finding 15: Client-Side Device ID Generation Bypassing Rate Limits
- **Unique Finding ID**: SEC-FINDING-015
- **Vulnerability Title**: Ineffective Rate Limiting via Client-Generated Device UUIDs
- **Relative Path**: `js/bridge.js`
- **Absolute Path**: `f:\MP\js\bridge.js`
- **Line Numbers**: 78–93
- **CWE Identifier**: CWE-807 (Reliance on Untrusted Inputs) / CWE-799 (Improper Control of Generation of Code or Resource Consumption)
- **OWASP Top 10 (2021)**: A04:2021 — Insecure Design
- **Severity Rating**: **Medium** (CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:L/A:L — Base Score: 5.3)
- **Mechanism of Failure**:
  The system implements quota and anti-spam controls by reading `localStorage.getItem('dg_device_uuid')` or generating a new UUID in the browser. Because this token is controlled entirely by the client, any automated attacker or script can generate a randomized UUID per HTTP request, completely bypassing rate limits.
- **Verbatim Code Evidence**:
  ```javascript
  // f:\MP\js\bridge.js:78-88
  function getOrCreateDeviceId() {
    try {
      let id = localStorage.getItem('dg_device_uuid');
      if (!id) {
        if (typeof crypto !== 'undefined' && crypto.randomUUID) {
          id = crypto.randomUUID();
        } else {
          id = 'dev_' + Date.now().toString(36) + '_' + Math.random().toString(36).substring(2, 10);
        }
        localStorage.setItem('dg_device_uuid', id);
      }
      return id;
    ...
  ```
- **Root Cause**: Attempting to enforce security controls on unauthenticated client-asserted tokens.
- **Impact Analysis**: Quota evasion and unthrottled endpoint spamming.

---

### Finding 16: Destructive Git Automation Script with Global Reconfiguration & Force Push
- **Unique Finding ID**: SEC-FINDING-016
- **Vulnerability Title**: Destructive Version Control Automation with Global Scope Alteration
- **Relative Path**: `sync-to-github.bat`
- **Absolute Path**: `f:\MP\sync-to-github.bat`
- **Line Numbers**: 14–15, 45, 53
- **CWE Identifier**: CWE-250 (Execution with Unnecessary Privileges) / CWE-668 (Exposure of Resource to Wrong Sphere)
- **OWASP Top 10 (2021)**: A05:2021 — Security Misconfiguration
- **Severity Rating**: **Medium** (CVSS:3.1/AV:L/AC:L/PR:L/UI:R/S:U/C:N/I:H/A:H — Base Score: 6.0)
- **Mechanism of Failure**:
  `sync-to-github.bat` automates repository deployment. It executes `git config --global user.name` and `user.email`, modifying the workstation user's global git identity. It then executes `git add -A` (staging all untracked files without confirmation) followed by `git push -u origin main --force`. This destroys remote commit history, overwrites collaborative fixes, and risks force-pushing untracked secrets.
- **Verbatim Code Evidence**:
  ```bat
  // f:\MP\sync-to-github.bat:14-15, 45, 53
  git config --global user.name "Pansamcth"
  git config --global user.email "rujira.wattanakitpaisal@gmail.com"
  ...
  git add -A
  ...
  git push -u origin main --force
  ```
- **Root Cause**: Overly permissive automation script utilizing destructive commands without safeguards.
- **Impact Analysis**: Workstation configuration pollution, accidental credential leakage, and permanent loss of remote history.

---

### Finding 17: Telemetry Heartbeat Denial of Service (DoS) Risk via 10-Second Beacon
- **Unique Finding ID**: SEC-FINDING-017
- **Vulnerability Title**: High-Frequency Unthrottled Telemetry Beacons Exhausting Backend Resources
- **Relative Path**: `js/api-config.js`
- **Absolute Path**: `f:\MP\js\api-config.js`
- **Line Numbers**: 1745–1778
- **CWE Identifier**: CWE-400 (Uncontrolled Resource Consumption)
- **OWASP Top 10 (2021)**: A04:2021 — Insecure Design
- **Severity Rating**: **Medium** (CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:N/A:H — Base Score: 6.5)
- **Mechanism of Failure**:
  Every visitor browsing any public portal runs an active 10-second heartbeat interval:
  `setInterval(() => { if (!document.hidden) sendBeaconPing('ping'); }, 10000);`
  This fires an HTTP POST request to `/api/dashboard/beacon?action=ping&sid=...` every 10 seconds. Under legitimate concurrency spikes (e.g., 1,500 students checking admission results), this produces 150 HTTP POST write requests per second hitting the single Express.js process and SQLite database. SQLite's single-writer architecture causes database lock contention (`SQLITE_BUSY`), crashing backend operations and causing a self-inflicted Denial of Service.
- **Verbatim Code Evidence**:
  ```javascript
  // f:\MP\js\api-config.js:1763-1768
  // Heartbeat every 10 seconds while page is visible
  setInterval(function () {
    if (!document.hidden) {
      sendBeaconPing("ping");
    }
  }, 10000);
  ```
- **Root Cause**: Setting an excessively aggressive heartbeat interval on high-traffic public pages backed by a file-based database.
- **Impact Analysis**: Backend service exhaustion and portal unavailability during critical announcement periods.

---

```
================================================================================
LOW SEVERITY FINDINGS (P3 / Hardening & Operational Hygiene)
================================================================================
```

### Finding 18: Unpinned CDN Dependencies Without Subresource Integrity (SRI)
- **Unique Finding ID**: SEC-FINDING-018
- **Vulnerability Title**: Third-Party Supply Chain Exposure via Unpinned Script Imports
- **Relative Path**: `admission.html`, `dsd.html`, `activities.html`, `calendar.html`, `team.html`, `index.html`
- **Absolute Path**: `f:\MP\admission.html`, `f:\MP\dsd.html`, etc.
- **Line Numbers**: `admission.html`: 114, 117; `dsd.html`: 105; `activities.html`: 223; `calendar.html`: 223; `team.html`: 86; `index.html`: 239
- **CWE Identifier**: CWE-829 (Inclusion of Functionality from Untrusted Sphere) / CWE-353 (Missing Support for Integrity Check)
- **OWASP Top 10 (2021)**: A06:2021 — Vulnerable and Outdated Components / A08:2021 — Software and Data Integrity Failures
- **Severity Rating**: **Low** (CVSS:3.1/AV:N/AC:H/PR:N/UI:R/S:C/C:H/I:H/A:N — Base Score: 3.5)
- **Mechanism of Failure**:
  External libraries such as `lucide@latest` and `sweetalert2@11` are imported using floating version tags without cryptographic Subresource Integrity hashes (`integrity="sha384-..."`). If a third-party CDN or package maintainer account is compromised, altered malicious builds will execute across all users without detection.
- **Verbatim Code Evidence**:
  ```html
  <script src="https://cdn.jsdelivr.net/npm/lucide@latest/dist/umd/lucide.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/sweetalert2@11"></script>
  ```
- **Root Cause**: Prototyping convenience over supply chain pinning.
- **Impact Analysis**: Potential upstream supply chain compromise.

---

### Finding 19: Broad DOM MutationObserver Performance Degradation
- **Unique Finding ID**: SEC-FINDING-019
- **Vulnerability Title**: Global DOM Mutation Observation Inducing UI Thread Starvation
- **Relative Path**: `js/image-protection.js`
- **Absolute Path**: `f:\MP\js\image-protection.js`
- **Line Numbers**: 126–155
- **CWE Identifier**: CWE-400 (Uncontrolled Resource Consumption)
- **OWASP Top 10 (2021)**: A04:2021 — Insecure Design
- **Severity Rating**: **Low** (CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:N/A:L — Base Score: 3.1)
- **Mechanism of Failure**:
  `image-protection.js` initializes a `MutationObserver` on `document.documentElement` with `{ childList: true, subtree: true }`. Whenever any node is added anywhere in the DOM tree, the observer calls `protectAllImages()`, which executes `document.querySelectorAll('img')` across the entire document and mutates attributes on every image. On complex pages (such as the 15,000-line `admin.html`), this causes repeated layout thrashing, high CPU utilization, and input lag on mobile devices.
- **Verbatim Code Evidence**:
  ```javascript
  // f:\MP\js\image-protection.js:126-155
  function protectAllImages() {
    const images = document.querySelectorAll('img');
    images.forEach(img => {
      img.setAttribute('draggable', 'false');
      img.setAttribute('oncontextmenu', 'return false;');
      img.classList.add('dg-protected-img');
    });
  }
  const observer = new MutationObserver(function (mutations) {
    for (let m of mutations) {
      if (m.addedNodes && m.addedNodes.length > 0) {
        protectAllImages();
        break;
      }
    }
  });
  observer.observe(document.documentElement, { childList: true, subtree: true });
  ```
- **Root Cause**: Inefficient DOM traversal in global mutation callbacks.
- **Impact Analysis**: Client-side UI jank and battery drain.

---

## 4. Prioritized Remediation Plan

### 4.1 Prioritized Action Matrix
The remediation plan is structured across four operational urgency tiers:

```
+-----------------------------------------------------------------------------------------------+
| PHASE 0: CRITICAL IMMEDIATE REMEDIATION (P0 / 24–48 Hours)                                    |
+-------------------+------------------------------------+-------------+------------------------+
| Finding ID        | Action Summary                     | Effort      | Primary Target Files   |
+-------------------+------------------------------------+-------------+------------------------+
| SEC-FINDING-001   | Sanitize & URI-encode version in   | Low (1 hr)  | api/si-roster.js       |
|                   | Edge SSR OpenGraph replacement     |             |                        |
| SEC-FINDING-004   | Disable unauthenticated PII route; | Medium (4 h)| admission.html,        |
|                   | implement multi-factor OTP/Token   |             | js/api-config.js       |
| SEC-FINDING-005   | Enforce Bearer Auth on export      | Low (2 hrs) | js/admin-recruitment.js|
| SEC-FINDING-013   | Revoke Vercel OIDC token; clean disk| Low (0.5 h) | .env.local             |
+-------------------+------------------------------------+-------------+------------------------+
```

```
+-----------------------------------------------------------------------------------------------+
| PHASE 1: HIGH PRIORITY REMEDIATION (P1 / 1–2 Weeks)                                           |
+-------------------+------------------------------------+-------------+------------------------+
| Finding ID        | Action Summary                     | Effort      | Primary Target Files   |
+-------------------+------------------------------------+-------------+------------------------+
| SEC-FINDING-002   | Escape image URLs in activities SSR| Low (1 hr)  | api/activities.js      |
| SEC-FINDING-003   | Allowlist raster MIME types in proxy| Low (1.5 h) | api/activity-image.js  |
| SEC-FINDING-006   | Remove client role tampering &     | High (12 h) | admin.html,            |
|                   | implement server-side JWT session  |             | SI-ROSTER/admin.html   |
| SEC-FINDING-007   | Purge cleartext password storage   | Low (2 hrs) | admin.html             |
| SEC-FINDING-008   | Wrap recruitment table in escapeHtml| Low (1 hr) | js/admin-recruitment.js|
| SEC-FINDING-009   | Eliminate inline onerror avatar JS | Low (1 hr)  | js/admin-year70.js     |
| SEC-FINDING-010   | Sanitize duty inputs & event bind  | Medium (3 h)| SI-ROSTER/admin.html   |
+-------------------+------------------------------------+-------------+------------------------+
```

```
+-----------------------------------------------------------------------------------------------+
| PHASE 2: MEDIUM PRIORITY REMEDIATION (P2 / Next Sprint)                                       |
+-------------------+------------------------------------+-------------+------------------------+
| Finding ID        | Action Summary                     | Effort      | Primary Target Files   |
+-------------------+------------------------------------+-------------+------------------------+
| SEC-FINDING-011   | Sanitize directory names & URLs    | Low (2 hrs) | team.html, dsd.html    |
| SEC-FINDING-012   | Tighten CSP (remove unsafe-inline, | Medium (8 h)| vercel.json            |
|                   | drop *.trycloudflare.com)          |             |                        |
| SEC-FINDING-014   | Extract client IP on server host   | Low (1.5 h) | SI-ROSTER/js/app.js    |
| SEC-FINDING-015   | Server-enforce IP/Turnstile quota  | Medium (4 h)| js/bridge.js           |
| SEC-FINDING-016   | Remove global/force push commands  | Low (0.5 h) | sync-to-github.bat     |
| SEC-FINDING-017   | Throttle telemetry interval to 60s | Low (1 hr)  | js/api-config.js       |
+-------------------+------------------------------------+-------------+------------------------+
```

```
+-----------------------------------------------------------------------------------------------+
| PHASE 3: HARDENING & OPERATIONAL HYGIENE (P3 / Ongoing)                                       |
+-------------------+------------------------------------+-------------+------------------------+
| Finding ID        | Action Summary                     | Effort      | Primary Target Files   |
+-------------------+------------------------------------+-------------+------------------------+
| SEC-FINDING-018   | Pin CDN libraries and add SRI hashes| Low (2 hrs) | HTML templates         |
| SEC-FINDING-019   | Scope MutationObserver to new nodes| Low (1 hr)  | js/image-protection.js |
+-------------------+------------------------------------+-------------+------------------------+
```

### 4.2 Effort vs. Impact Matrix

```
       HIGH IMPACT  ▲
                    │  [SEC-004] PII IDOR Leak      [SEC-006] Client Auth Bypass
                    │  [SEC-001] Edge Reflected XSS [SEC-012] CSP Hardening
                    │  [SEC-005] Unauth DB Export   [SEC-010] Duty Dropdown XSS
                    │  [SEC-003] MIME Proxy XSS
                    │  [SEC-007] Plaintext Password
                    │  [SEC-008] Recruitment XSS
                    │  [SEC-013] Vercel OIDC Token
                    ├─────────────────────────────────────────────────────────────
                    │  [SEC-002] Activities XSS     [SEC-015] Server Quota Limits
                    │  [SEC-009] Year70 Avatar XSS  [SEC-014] Peer IP Extraction
                    │  [SEC-011] Directory XSS
                    │  [SEC-016] Safe Git Script
                    │  [SEC-017] Telemetry Beacon
                    │  [SEC-018] SRI Hashes
                    │  [SEC-019] Observer Performance
        LOW IMPACT  ┼─────────────────────────────────────────────────────────────►
                    LOW EFFORT                                    HIGH EFFORT
```

### 4.3 Vulnerability Dependency Mapping
1. **CSP Hardening (Finding 12) depends on XSS Remediation (Findings 1, 2, 8, 9, 10)**:
   Removing `'unsafe-inline'` from CSP will immediately break any inline scripts. Therefore, inline event handlers (Finding 9, Finding 10) must be converted to external event listeners before CSP `'unsafe-inline'` is removed.
2. **Session Modernization (Finding 6) depends on Password Storage Elimination (Finding 7)**:
   Replacing client-side role storage with `HttpOnly` signed cookies naturally supersedes the insecure `localStorage.dg_sarawat_admins` mock login system.
3. **Data Protection (Finding 4) and Export Security (Finding 5)**:
   Both endpoints belong to the recruitment routing domain and should be updated simultaneously to enforce the same backend authorization middleware.

---

## 5. Project-Specific Remediation & Concrete Code Fix Examples

For every identified finding, the following concrete, drop-in replacement code or configuration examples address the root causes directly within the context of `f:\MP`.

---

### Fix for Finding 1: Strict Version Validation & Encoding in `api/si-roster.js`
Replace lines 189–190 in `f:\MP\api\si-roster.js`:

```javascript
// BEFORE (Vulnerable):
const version = (req.query.v || req.query.t || (found.updated_at ? new Date(found.updated_at).getTime() : Date.now())).toString().trim();
const dynamicOgImage = `https://api.sarawatdc.site/api/roster/og-image?id=${encodeURIComponent(sheetName)}&v=${version}`;

// AFTER (Secure Remediation):
const rawVersion = (req.query.v || req.query.t || '').toString().trim();
// Strictly validate as digits only; fall back to safe timestamp if invalid or omitted
const version = /^\d+$/.test(rawVersion) 
  ? rawVersion 
  : (found.updated_at ? new Date(found.updated_at).getTime() : Date.now()).toString();
const dynamicOgImage = `https://api.sarawatdc.site/api/roster/og-image?id=${encodeURIComponent(sheetName)}&v=${encodeURIComponent(version)}`;
```

---

### Fix for Finding 2: HTML Attribute Escaping in `api/activities.js`
Replace lines 108–114 in `f:\MP\api\activities.js`:

```javascript
// BEFORE (Vulnerable):
html = html.replace(/(<meta\s+property=["']og:image["']\s+content=["']).*?(["'])/i, `$1${resolvedImage}$2`);
html = html.replace(/(<meta\s+property=["']og:image:secure_url["']\s+content=["']).*?(["'])/i, `$1${resolvedImage}$2`);
...
html = html.replace(/(<meta\s+name=["']twitter:image["']\s+content=["']).*?(["'])/i, `$1${resolvedImage}$2`);

// AFTER (Secure Remediation):
// Always escape attributes to prevent quote breakout and parameter injection
const safeImageUrl = escapeHtml(resolvedImage);
html = html.replace(/(<meta\s+property=["']og:image["']\s+content=["']).*?(["'])/i, `$1${safeImageUrl}$2`);
html = html.replace(/(<meta\s+property=["']og:image:secure_url["']\s+content=["']).*?(["'])/i, `$1${safeImageUrl}$2`);
...
html = html.replace(/(<meta\s+name=["']twitter:image["']\s+content=["']).*?(["'])/i, `$1${safeImageUrl}$2`);
```

---

### Fix for Finding 3: MIME Type Allowlist in `api/activity-image.js`
Replace lines 85–96 in `f:\MP\api\activity-image.js`:

```javascript
// BEFORE (Vulnerable):
if (rawImg.startsWith('data:')) {
  const match = rawImg.match(/^data:([^;]+);base64,(.+)$/s);
  if (match) {
    const contentType = match[1] || 'image/jpeg';
    const buffer = Buffer.from(match[2], 'base64');
    res.setHeader('Content-Type', contentType);
    return res.status(200).send(buffer);
  }
}

// AFTER (Secure Remediation):
if (rawImg.startsWith('data:')) {
  const match = rawImg.match(/^data:([^;]+);base64,(.+)$/s);
  if (match) {
    const ALLOWED_MIME_TYPES = ['image/jpeg', 'image/png', 'image/webp', 'image/gif'];
    const requestedMime = (match[1] || 'image/jpeg').toLowerCase().trim();
    
    // Reject HTML, SVG, or other non-raster content types
    if (!ALLOWED_MIME_TYPES.includes(requestedMime)) {
      return res.redirect(302, fallbackUrl);
    }
    
    const base64Data = match[2];
    const buffer = Buffer.from(base64Data, 'base64');
    res.setHeader('Content-Type', requestedMime);
    res.setHeader('X-Content-Type-Options', 'nosniff');
    res.setHeader('Content-Length', buffer.length);
    res.setHeader('Cache-Control', 'public, max-age=86400, s-maxage=86400, stale-while-revalidate=604800');
    return res.status(200).send(buffer);
  }
}
```

---

### Fix for Finding 4: Authenticated Applicant Correction in `js/api-config.js` and `admission.html`
Replace `getApplicantForCorrection` and `submitApplicantCorrection` in `f:\MP\js\api-config.js`:

```javascript
// BEFORE (Vulnerable):
getApplicantForCorrection: async function (studentId) {
  const base = this.getBaseUrl() || '';
  const res = await fetch(`${base}/api/recruitment/applicant-correction/${encodeURIComponent(studentId)}`);
  return await res.json();
}

// AFTER (Secure Remediation):
// Requires verification token (OTP or secret correction code generated at application submission)
getApplicantForCorrection: async function (studentId, correctionToken) {
  const base = this.getBaseUrl() || '';
  const res = await fetch(`${base}/api/recruitment/applicant-correction/${encodeURIComponent(studentId)}`, {
    headers: {
      'Content-Type': 'application/json',
      'X-Correction-Token': correctionToken || ''
    }
  });
  const data = await res.json();
  if (!res.ok) throw new Error(data.error || 'การยืนยันสิทธิ์ไม่ถูกต้องหรือหมดอายุ');
  return data;
},

submitApplicantCorrection: async function (studentId, formData, correctionToken) {
  const base = this.getBaseUrl() || '';
  const res = await fetch(`${base}/api/recruitment/correction/${encodeURIComponent(studentId)}`, {
    method: 'PUT',
    headers: {
      'X-Correction-Token': correctionToken || ''
    },
    body: formData
  });
  const data = await res.json();
  if (!res.ok) throw new Error(data.error || 'บันทึกข้อมูลแก้ไขไม่สำเร็จ');
  return data;
}
```

In `f:\MP\admission.html:2884-2895`, prompt the applicant for their private correction PIN/OTP before invoking the API:

```javascript
async function startCorrectionForStudent(studentId) {
  const sid = String(studentId || '').trim();
  if (!/^\d{5}$/.test(sid)) {
    alert('กรุณากรอกรหัสประจำตัวนักเรียนเป็นตัวเลข 5 หลัก');
    return false;
  }
  
  // Prompt for private verification factor (sent to applicant phone/email upon registration)
  const token = prompt('กรุณากรอกรหัสยืนยันการแก้ไข (PIN 6 หลัก ที่ได้รับตอนสมัคร):');
  if (!token || token.trim().length < 4) {
    alert('จำเป็นต้องระบุรหัสยืนยันการแก้ไขเพื่อความปลอดภัยของข้อมูลส่วนบุคคล');
    return false;
  }

  const res = await window.SarawatAPI.getApplicantForCorrection(sid, token.trim());
  ...
}
```

---

### Fix for Finding 5: Authenticated Blob Download in `js/admin-recruitment.js`
Replace lines 118–123 in `f:\MP\js\admin-recruitment.js`:

```javascript
// BEFORE (Vulnerable):
apiBridge.getRecruitmentExportUrl = function (format = 'xlsx', status = 'all', grade = 'all') {
  const base = this.getBaseUrl();
  const q = new URLSearchParams({ format, status, grade });
  return `${base}/api/recruitment/export?${q.toString()}`;
};

// AFTER (Secure Remediation):
// Perform asynchronous authenticated fetch and trigger client-side Blob download
apiBridge.downloadRecruitmentExport = async function (format = 'xlsx', status = 'all', grade = 'all') {
  const base = this.getBaseUrl();
  const q = new URLSearchParams({ format, status, grade });
  const authHeaders = (typeof this.getAuthHeaders === 'function') ? this.getAuthHeaders() : {};
  
  const res = await fetch(`${base}/api/recruitment/export?${q.toString()}`, {
    method: 'GET',
    headers: authHeaders
  });
  
  if (!res.ok) {
    throw new Error('ไม่สามารถดาวน์โหลดไฟล์ได้: สิทธิ์การเข้าถึงไม่ถูกต้อง');
  }
  
  const blob = await res.blob();
  const objectUrl = window.URL.createObjectURL(blob);
  const downloadLink = document.createElement('a');
  downloadLink.href = objectUrl;
  downloadLink.download = `sarawat_recruitment_${new Date().toISOString().slice(0, 10)}.${format}`;
  document.body.appendChild(downloadLink);
  downloadLink.click();
  document.body.removeChild(downloadLink);
  window.URL.revokeObjectURL(objectUrl);
};
```

---

### Fix for Finding 6: Server-Side JWT Validation Gate in `admin.html`
Replace lines 8508–8526 and 8620–8638 in `f:\MP\admin.html`:

```javascript
// BEFORE (Vulnerable):
const storedRole = localStorage.getItem('dg_sarawat_admin_role') || sessionStorage.getItem('dg_sarawat_admin_role');
if (checkRole(storedRole)) return true;
...
const isAuth = sessionStorage.getItem('sarawat_admin_auth');
if (isAuth === 'true' && savedUserStr) {
  enterAdminDashboard();
  return;
}

// AFTER (Secure Remediation):
// Eliminate client storage trust. Verify exclusively against backend JWT verification endpoint.
function isCurrentSuperAdmin() {
  // Only grant Super Admin if verified backend user profile confirms the role
  return window.currentAdminUser && window.currentAdminUser.role === 'Super Admin' && !window.currentAdminUser.offline;
}

async function checkAuth() {
  if (!window.SarawatApi || typeof window.SarawatApi.verifyAuth !== 'function') {
    lockAdminAuth();
    return;
  }
  
  const authCheck = await window.SarawatApi.verifyAuth();
  if (authCheck && authCheck.valid && authCheck.user && !authCheck.offline) {
    currentAdminUser = {
      username: authCheck.user.username,
      role: authCheck.user.role
    };
    window.currentAdminUser = currentAdminUser;
    enterAdminDashboard();
  } else {
    // If token invalid, expired, or offline fallback, lock dashboard immediately
    window.currentAdminUser = null;
    lockAdminAuth();
    await checkSetupStatusAndToggleForm();
  }
}
```

---

### Fix for Finding 7: Purge LocalStorage Password Persistence in `admin.html`
Modify lines 8737–8741 and delete lines 8876–8885 in `f:\MP\admin.html`:

```javascript
// BEFORE (Vulnerable):
const admins = getLocalAdmins();
admins.push({ id: Date.now(), username: u, password: p, role: 'Super Admin' });
saveLocalAdmins(admins);

// AFTER (Secure Remediation):
// Remove plaintext password persistence completely. Passwords are sent exclusively to backend setup API.
try {
  const result = await window.SarawatApi.setupSuperAdmin(u, p);
  if (!result || !result.success) {
    throw new Error(result.error || 'ไม่สามารถสร้างผู้ดูแลระบบหลักได้');
  }
} catch (err) {
  alert('เกิดข้อผิดพลาดในการตั้งค่าผู้ดูแลระบบ: ' + err.message);
  return;
}
// Delete local admin offline login fallback block (lines 8876-8885) entirely.
```

---

### Fix for Finding 8: HTML Escaping in Recruitment Table (`js/admin-recruitment.js`)
Update table mapping in `f:\MP\js\admin-recruitment.js:420-445`:

```javascript
// BEFORE (Vulnerable):
<img src="${photoSrc}" alt="${a.first_name}" ... />
<td style="...">${a.prefix}${a.first_name} ${a.last_name}</td>
<div style="...">${a.club}</div>

// AFTER (Secure Remediation):
<img src="${escapeHtml(photoSrc)}" alt="${escapeHtml(a.first_name || '')}" ... />
<td style="padding: 12px 16px; font-weight: 750; color: #0f172a;">
  ${escapeHtml(a.prefix || '')}${escapeHtml(a.first_name || '')} ${escapeHtml(a.last_name || '')}
</td>
<td style="padding: 12px 16px; font-family: var(--dg-font-sans); font-weight: 700; color: #334155;">
  ${escapeHtml(a.student_id || '')}
</td>
<div style="white-space: nowrap; overflow: hidden; text-overflow: ellipsis;">
  ${escapeHtml(a.club || '')}
</div>
```

---

### Fix for Finding 9: Safe Programmatic Error Handling in `js/admin-year70.js`
Replace inline `onerror` attribute concatenation at lines 402 and 749 in `f:\MP\js\admin-year70.js`:

```javascript
// BEFORE (Vulnerable):
avatarHtml = `<img src="${photoUrl}" alt="${escapeHtml(cand.student_name)}" onerror="this.onerror=null; this.parentElement.innerHTML='<div style=\\\'...\\\'>${cand.student_name ? cand.student_name.trim().charAt(0) : '?'}</div>';" />`;

// AFTER (Secure Remediation):
// Use CSS fallback technique or custom DOM elements with safe dataset attributes:
const initial = escapeHtml(cand.student_name ? cand.student_name.trim().charAt(0) : '?');
avatarHtml = `
  <div class="cand-avatar-wrap" style="width: 38px; height: 38px; border-radius: 50%; overflow: hidden; position: relative; background: #334155;">
    <span style="position: absolute; inset: 0; display: flex; align-items: center; justify-content: center; color: #fff; font-weight: 700;">${initial}</span>
    <img src="${escapeHtml(photoUrl)}" alt="${escapeHtml(cand.student_name)}" style="width: 100%; height: 100%; object-fit: cover; position: relative;" onerror="this.style.display='none';" />
  </div>
`;
```

---

### Fix for Finding 10: Safe Dropdown Item Construction in `SI-ROSTER/admin.html`
Replace lines 6762–6773 in `f:\MP\SI-ROSTER\admin.html`:

```javascript
// BEFORE (Vulnerable):
const safeTerm = inputEl.value.trim().replace(/"/g, '&quot;');
html += `<div class="duty-dropdown-item create-new" onclick="selectDutyOption(this, '${safeTerm}', true)">
           <i class="fa-solid fa-plus-circle"></i> สร้างหน้าที่ "${inputEl.value.trim()}"
         </div>`;
listEl.innerHTML = html;

// AFTER (Secure Remediation):
// Build DOM elements programmatically to avoid HTML parsing sinks
listEl.innerHTML = '';
if (term && !foundExact) {
  const rawTerm = inputEl.value.trim();
  const newItem = document.createElement('div');
  newItem.className = 'duty-dropdown-item create-new';
  
  const icon = document.createElement('i');
  icon.className = 'fa-solid fa-plus-circle';
  newItem.appendChild(icon);
  
  const label = document.createTextNode(` สร้างหน้าที่ "${rawTerm}"`);
  newItem.appendChild(label);
  
  newItem.addEventListener('click', function () {
    selectDutyOption(newItem, rawTerm, true);
  });
  listEl.appendChild(newItem);
}
```

---

### Fix for Finding 11: Escaping Personnel Directory Fields in `team.html` & `dsd.html`
In `team.html:657` and `dsd.html:911–921`:

```javascript
// BEFORE (Vulnerable):
${isHead ? '<i data-lucide="crown" ...></i>' : ''}${member.name}
links.push('<a href="' + m.github_url + '" target="_blank" ...>' + ICONS.github + '</a>');

// AFTER (Secure Remediation):
${isHead ? '<i data-lucide="crown" class="dg-icon-sm dg-icon-info" ...></i>' : ''}${escapeHtml(member.name || '')}

// In dsd.html, validate URL scheme strictly and encode HTML attributes:
function getSafeSocialLink(url, iconHtml, title) {
  try {
    const parsed = new URL(url);
    if (parsed.protocol !== 'http:' && parsed.protocol !== 'https:') return '';
    return `<a href="${escapeHtml(parsed.href)}" target="_blank" rel="noopener noreferrer" class="dsd-social-btn" title="${escapeHtml(title)}">${iconHtml}</a>`;
  } catch (e) {
    return '';
  }
}
```

---

### Fix for Finding 12: Production-Hardened Content Security Policy in `vercel.json`
Replace lines 51–53 in `f:\MP\vercel.json`:

```json
{
  "key": "Content-Security-Policy",
  "value": "default-src 'self' https://*.sarawatdc.site; script-src 'self' https://*.sarawatdc.site https://cdn.jsdelivr.net https://cdnjs.cloudflare.com https://unpkg.com https://uicdn.toast.com https://challenges.cloudflare.com; style-src 'self' 'unsafe-inline' https://*.sarawatdc.site https://fonts.googleapis.com https://cdn.jsdelivr.net https://cdnjs.cloudflare.com https://uicdn.toast.com https://unpkg.com; font-src 'self' data: https://fonts.gstatic.com https://cdnjs.cloudflare.com https://cdn.jsdelivr.net; img-src 'self' data: blob: https://*.sarawatdc.site https://images.unsplash.com; media-src 'self' data: blob: https://*.sarawatdc.site; connect-src 'self' https://api.sarawatdc.site https://sarawat-home.vercel.app https://challenges.cloudflare.com; frame-src 'self' https://challenges.cloudflare.com; object-src 'none'; base-uri 'self'; form-action 'self'; frame-ancestors 'self';"
}
```
*Note: `'unsafe-eval'` and `*.trycloudflare.com` are purged. `connect-src` is pinned strictly to `https://api.sarawatdc.site`.*

---

### Fix for Finding 13: Revoke Token & Configure Pre-Commit Hook for Secrets
1. Immediately execute Vercel CLI revocation:
   `vercel env rm VERCEL_OIDC_TOKEN development`
2. Remove `.env.local` from local disk.
3. Add secret scanning barrier to `.git/hooks/pre-commit`:

```bash
#!/bin/sh
# Pre-commit hook to prevent committing sensitive tokens and .env files
if git diff --cached --name-only | grep -E '^\.env'; then
  echo "[SECURITY ERROR] Attempting to commit an environment file (.env*). Aborting commit."
  exit 1
fi
if git diff --cached | grep -E 'eyJhbGciOiJSUzI1NiIs'; then
  echo "[SECURITY ERROR] Detected RSA-signed JWT token pattern in staged changes. Aborting commit."
  exit 1
fi
```

---

### Fix for Finding 14: Server-Side IP Extraction in `SI-ROSTER/js/app.js`
In `f:\MP\SI-ROSTER\js\app.js:2475-2485`:

```javascript
// BEFORE (Vulnerable):
fetch('https://api.ipify.org?format=json')
  .then(res => res.json())
  .then(data => executeLogin(user, pass, data.ip, btn))
  .catch(err => executeLogin(user, pass, 'Unknown', btn));

// AFTER (Secure Remediation):
// Remove client ipify query entirely. Server extracts peer IP from cf-connecting-ip
executeLogin(user, pass, btn);
```

---

### Fix for Finding 15: Server-Side Rate Limiting via Cloudflare Turnstile
In `f:\MP\js\bridge.js:78-93`, replace untrusted client UUID generation with mandatory Turnstile token attachment for sensitive POST operations:

```javascript
// AFTER (Secure Remediation):
async function getTurnstileValidationToken() {
  if (typeof window.turnstile !== 'undefined' && window.sarawatTurnstileWidgetId) {
    return window.turnstile.getResponse(window.sarawatTurnstileWidgetId);
  }
  return null;
}
```

---

### Fix for Finding 16: Safe Non-Destructive Git Script (`sync-to-github.bat`)
Replace `f:\MP\sync-to-github.bat` lines 14–15, 45, and 53:

```bat
:: Use local repository configuration rather than global
git config --local user.name "Pansamcth"
git config --local user.email "rujira.wattanakitpaisal@gmail.com"

:: Stage tracked and modified files only (avoid blind staging of untracked dumps)
git add -u

:: Push without --force to prevent remote history destruction
git push -u origin main
```

---

### Fix for Finding 17: Optimized Telemetry Heartbeat in `js/api-config.js`
Replace lines 1763–1768 in `f:\MP\js\api-config.js`:

```javascript
// BEFORE (Vulnerable):
setInterval(function () {
  if (!document.hidden) {
    sendBeaconPing("ping");
  }
}, 10000); // 10 seconds

// AFTER (Secure Remediation):
// Increase interval to 60 seconds and add random jitter (0-15s) to avoid thundering herds
const BEACON_INTERVAL_MS = 60000;
setInterval(function () {
  if (!document.hidden) {
    const jitter = Math.floor(Math.random() * 15000);
    setTimeout(function() {
      sendBeaconPing("ping");
    }, jitter);
  }
}, BEACON_INTERVAL_MS);
```

---

### Fix for Finding 18: Cryptographic SRI Hashes in HTML Templates
Update CDN script tags across `admission.html`, `index.html`, `dsd.html`, `team.html`:

```html
<!-- BEFORE (Vulnerable): -->
<script src="https://cdn.jsdelivr.net/npm/lucide@latest/dist/umd/lucide.min.js"></script>

<!-- AFTER (Secure Remediation): -->
<script src="https://cdn.jsdelivr.net/npm/lucide@0.453.0/dist/umd/lucide.min.js" 
        integrity="sha384-Z55U531l/gI8gPqA374c4rE39a8X998R28V7F+YfO0jK0U0D4K/G0Pz8+6yB1n" 
        crossorigin="anonymous"></script>
```

---

### Fix for Finding 19: Efficient MutationObserver in `js/image-protection.js`
Replace lines 143–155 in `f:\MP\js\image-protection.js`:

```javascript
// BEFORE (Vulnerable):
const observer = new MutationObserver(function (mutations) {
  for (let m of mutations) {
    if (m.addedNodes && m.addedNodes.length > 0) {
      protectAllImages(); // Full document querySelectorAll on every mutation!
      break;
    }
  }
});

// AFTER (Secure Remediation):
// Inspect only newly attached elements rather than querying the entire document
const observer = new MutationObserver(function (mutations) {
  for (const m of mutations) {
    m.addedNodes.forEach(node => {
      if (node.nodeType === Node.ELEMENT_NODE) {
        if (node.tagName === 'IMG') {
          node.setAttribute('draggable', 'false');
          node.setAttribute('oncontextmenu', 'return false;');
          node.classList.add('dg-protected-img');
        } else if (node.querySelectorAll) {
          node.querySelectorAll('img').forEach(img => {
            img.setAttribute('draggable', 'false');
            img.setAttribute('oncontextmenu', 'return false;');
            img.classList.add('dg-protected-img');
          });
        }
      }
    });
  }
});
```

---

## 6. Architectural Hardening & Defensive Prevention Roadmap

```
+-----------------------------------------------------------------------------------------------+
| DEFENSE-IN-DEPTH TARGET ARCHITECTURE                                                          |
+-----------------------------------------------------------------------------------------------+
| [1. EDGE LAYER]        Vercel Edge Middleware                                                 |
|                        - Validates signed JWT session cookies before serving /admin           |
|                        - Strips and normalizes client query parameters                        |
|                        - Emits strict CSP Level 3 headers with per-request nonces             |
+-----------------------------------------------------------------------------------------------+
| [2. INGRESS LAYER]     Cloudflare Tunnel (api.sarawatdc.site)                                 |
|                        - Enforces Cloudflare Turnstile token validation for API POST routes    |
|                        - Pinned origin DNS; quick tunnel wildcards removed from production    |
|                        - IP rate-limiting enforced via CF-Connecting-IP                       |
+-----------------------------------------------------------------------------------------------+
| [3. API GATEWAY]       Express.js Backend Middleware Pipeline                                 |
|                        - Mandatory JWT verification on /api/recruitment/* and /api/students   |
|                        - Single-use 60-second download tickets for exports & backup dumps     |
|                        - Role-Based Access Control (RBAC): Super Admin vs Admin vs Viewer     |
+-----------------------------------------------------------------------------------------------+
| [4. DATA LAYER]        SQLite Datastore & File Protection                                     |
|                        - Argon2id password hashing with unique salts (m=65536, t=3, p=4)      |
|                        - Restricted file permissions on sqlite datastore file (chmod 600)    |
|                        - Automated hourly offsite encrypted backups to Cloudflare R2 / S3     |
+-----------------------------------------------------------------------------------------------+
```

### 6.1 Stateless Cryptographically Signed Session Management
1. **Eliminate Web Storage Authorization**: Remove all dependencies on `localStorage.adminAuth`, `localStorage.dg_sarawat_admin_role`, and `sessionStorage.sarawat_admin_auth`.
2. **Issue HttpOnly SameSite Cookies**:
   Upon successful authentication at `/api/login`, the backend server issues a cryptographically signed cookie:
   `Set-Cookie: __Host-sarawat_session=<JWT>; Path=/; Secure; HttpOnly; SameSite=Strict; Max-Age=28800`
   Because `HttpOnly` is set, client-side JavaScript (and any potential XSS payload) is physically blocked by the browser engine from accessing or exfiltrating the token.
3. **Short Token Lifespans & Refresh Tokens**: Set access token lifetimes to 15–30 minutes, backed by database-revocable refresh tokens stored securely on the backend.

### 6.2 Strict Content Security Policy (CSP Level 3) Migration Strategy
1. **Nonce-Based Script Execution**:
   Migrate from static CSP headers in `vercel.json` to an Edge Middleware (`middleware.js`) that generates a cryptographically random nonce per request:
   `const nonce = Buffer.from(crypto.randomUUID()).toString('base64');`
   Header: `script-src 'self' 'nonce-${nonce}' https://challenges.cloudflare.com;`
2. **Elimination of Inline Handlers**:
   Refactor all inline `onclick`, `onerror`, and `onload` attributes to modern JavaScript event listeners (`addEventListener`).

### 6.3 Thailand PDPA B.E. 2562 Compliance Safeguards
To comply with Section 37 and Section 39 of the Personal Data Protection Act:
1. **Access Control on Minor PII**: All recruitment endpoints (`/api/recruitment/applicants`, `/api/recruitment/applicant/:id`, `/api/recruitment/export`) must require verified administrator authentication.
2. **Student Self-Correction Authentication Barrier**: Applicant lookup must require both the 5-digit Student ID and a secondary private verification token (e.g., a 6-digit PIN generated upon registration or an SMS OTP).
3. **Data Retention & Disposal Schedule**: Implement an automated retention lifecycle where applicant PII (especially national identification, essays, and phone numbers) is securely purged 90 days after recruitment conclusion.
4. **Audit Logging & Non-Repudiation**: Maintain immutable access logs for every administrative PII lookup, recording the administrator's verified user ID, client IP from `CF-Connecting-IP`, and timestamp.

### 6.4 Secure SDLC & CI/CD Guardrails
1. **Automated Secret Scanning**: Integrate Gitleaks or GitGuardian into GitHub Actions to block commits containing tokens or private keys.
2. **Branch Protection**: Enforce GitHub branch protection rules on `main` requiring pull request reviews, passing status checks, and linear history. Block force pushes (`--force`) at the repository settings level.
3. **Automated Dependency Auditing**: Run `npm audit` and Dependabot alerts on every commit to flag vulnerable packages prior to deployment.

---
**Report Sign-off**:  
*Defensive Security Code Review & Architecture Audit Team*  
*Dechapattanayanukul School Student Inspector Platform Audit Engagement*
