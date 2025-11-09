# VAPT-Report-for-Web-Server-with-Live-Website-
A Vulnerability Assessment and Penetration Testing (VAPT) report for a live web server (http://10.91.141.162:8080 ) was conducted using ZAP by Checkmarx. The scan identified missing security headers (CSP, Anti-clickjacking, X-Content-Type-Options), vulnerable JS libraries, and minor information disclosure issues.

# VAPT Report — Live Web Server (10.91.141.162:8080)

> **Assessment type:** External web application vulnerability scan (black-box)
> **Target:** `http://10.91.141.162:8080`
> **Scanner / Report:** ZAP by Checkmarx (ZAP v2.16.1)
> **Scan date:** 27 Aug 2025
> **Generated:** 27 Aug 2025, 22:19:37

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Scope & Methodology](#scope--methodology)
3. [Summary of Findings](#summary-of-findings)
4. [Detailed Findings (priority order)](#detailed-findings-priority-order)
5. [Remediation & Best Practices](#remediation--best-practices)
6. [Scan Statistics & Appendix](#scan-statistics--appendix)
7. [Next Steps](#next-steps)

---

## Executive Summary

A web application scan was performed against `http://10.91.141.162:8080` using ZAP. The scan identified multiple client-side security header omissions, outdated third-party JavaScript assets, and other information disclosure issues that increase the attack surface. No confirmed SQLi or inline XSS payloads were found in the URL column of the exported logs, but a number of endpoints (file upload, contact/captcha, admin) warrant manual review and hardening.

**Overall risk posture:** *Medium* — actionable issues with moderate impact (client-side attacks, third-party component vulnerabilities, and information disclosure).

---

## Scope & Methodology

**In scope**

* Target host: `10.91.141.162:8080`
* Web application endpoints discovered by the scanner (including static assets)
* Passive and active checks by ZAP (default + passive scanners)

**Out of scope**

* Authenticated testing (no credentials provided)
* Full source-code review
* Internal network / host-level testing

**Methodology**

* Automated scanning with OWASP ZAP v2.16.1 (passive + active rules)
* Review of HTTP responses, headers, and referenced libraries
* Aggregation of scanner alerts into a prioritized findings list

---

## Summary of Findings

|      Severity | Issue                                                                                          | Count / Notes                                                                   |
| ------------: | ---------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
|        Medium | **Content-Security-Policy (CSP) header not set**                                               | Multiple endpoints (e.g., `/admin/en/forgot-password`, `/tl/vulnerability?...`) |
|        Medium | **Missing Anti-clickjacking header** (`X-Frame-Options` / `frame-ancestors`)                   | Observed on root and multiple pages                                             |
|           Low | **X-Content-Type-Options header missing** (`nosniff`)                                          | Observed                                                                        |
|        Medium | **Vulnerable / outdated JS libraries** (jquery 3.4.1, bootstrap, Magnific-Popup, owl-carousel) | Multiple references; requires version checks                                    |
|           Low | **Cookies without `SameSite` attribute**                                                       | Observed                                                                        |
| Informational | **Sensitive information in URL / suspicious comments**                                         | Some endpoints expose query strings with user input (contact form tests)        |
|   Info/Medium | **File upload endpoints present**                                                              | Endpoints contain `upload` / multipart patterns — verify server-side validation |
| Informational | **Exposed admin paths**                                                                        | `admin` pages accessible — require access controls                              |

> ZAP alert summary in report: Risk counts included Medium, Low, and Informational alerts across the scanned site (see Scan Statistics).

---

## Detailed Findings (priority order)

### Finding 1 — Content Security Policy (CSP) Header Not Set

* **Severity:** Medium
* **Affected endpoints (examples):**

  * `http://10.91.141.162:8080/admin/en/forgot-password`
  * `http://10.91.141.162:8080/tl/vulnerability?captcha=ZAP&email=...`
* **Description:** No `Content-Security-Policy` header present. Without CSP, the browser cannot restrict script/style sources, increasing XSS and content injection risk.
* **Impact:** Theft of user tokens, session hijacking, client-side malware injection.
* **Remediation:** Implement a restrictive CSP; test in `report-only` mode first. Example:

  ```
  Content-Security-Policy: default-src 'self'; script-src 'self' https://trusted.cdn.example; style-src 'self'; frame-ancestors 'none'; object-src 'none';
  ```

---

### Finding 2 — Missing Anti-Clickjacking Header

* **Severity:** Medium
* **Description:** `X-Frame-Options` or `Content-Security-Policy: frame-ancestors` absent.
* **Impact:** Page can be embedded in attacker-controlled frames facilitating clickjacking.
* **Remediation:** Add `X-Frame-Options: DENY` or `Content-Security-Policy: frame-ancestors 'none'`. Use `DENY` or `SAMEORIGIN` based on legitimate framing needs.

---

### Finding 3 — Vulnerable / Outdated JavaScript Libraries

* **Severity:** Medium → High (context dependent)
* **Examples observed:** `jquery.3.4.1.js`, `bootstrap.min.js`, Magnific-Popup, owl-carousel.
* **Description:** Older library versions may contain known CVEs that are trivially exploited via RCE, XSS, or prototype pollution.
* **Remediation:** Inventory library versions; upgrade to supported versions; apply mitigation if immediate upgrade not possible (e.g., CSP, subresource integrity, monitor for exploit attempts).

---

### Finding 4 — X-Content-Type-Options Missing, Cookies Missing SameSite

* **Severity:** Low → Medium combined impact
* **Description:** `X-Content-Type-Options: nosniff` missing (risk of MIME sniffing). Cookies missing `SameSite` can be abused in CSRF scenarios.
* **Remediation:** Add header `X-Content-Type-Options: nosniff`. Set cookie attributes `HttpOnly; Secure; SameSite=Lax/Strict` as appropriate.

---

### Finding 5 — File Upload / Form Endpoints & Sensitive Input in URLs

* **Severity:** Medium for upload endpoints; Informational for contact params
* **Description:** Several URLs include `upload`, `captcha`, `message`, `email` parameters — these indicate file upload and form handlers which often accept user-generated content.
* **Remediation:** Enforce server-side validation, content-type checks, virus scanning for uploads, storage outside webroot, and limit filename handling; sanitize and validate all form inputs on server side.

---

## Remediation & Best Practices (recommended priority)

1. **CSP + frame protection** — Implement and test Content-Security-Policy and `frame-ancestors` / `X-Frame-Options`. (High)
2. **Patch third-party libs** — Upgrade jQuery, Bootstrap, and other vendor assets. Check CVE databases for versions in use. (High)
3. **HTTP security headers** — `X-Content-Type-Options: nosniff`, `Referrer-Policy`, `Permissions-Policy`, `Strict-Transport-Security` if HTTPS used. (Medium)
4. **Cookie hardening** — `Secure`, `HttpOnly`, `SameSite` attributes for all session cookies. (Medium)
5. **Harden upload endpoints** — Validate file types, size limits, store safely, virus scanning. (High for any upload endpoints)
6. **Protect admin interfaces** — Require authentication, IP whitelisting, rate-limiting, enable MFA for admin users. (High)
7. **Logging & monitoring** — Centralized logs, WAF rules for suspicious patterns, IDS/IPS adjustments. (Medium)
8. **Follow-up testing** — Perform authenticated scans, manual verification of Medium/High findings, and targeted tests for file upload and admin endpoints. (High)

---

## Scan Statistics & Appendix

* **ZAP version:** 2.16.1
* **Report generation timestamp:** Wed 27 Aug 2025, 22:19:37
* **Alert distribution (high-level):** Medium, Low, Informational alerts present. ZAP output included:

  * **Content Security Policy (CSP) Header Not Set** — multiple alerts
  * **Missing Anti-clickjacking Header** — multiple alerts
  * **Vulnerable JS Library** — occurrences (e.g., `bootstrap.min.js`, `jquery.3.4.1.js`)
  * **Cookie without SameSite Attribute** — occurrences
  * **X-Content-Type-Options Header Missing** — occurrences
  * **Information disclosure (sensitive in URL / suspicious comments)** — occurrences


---
