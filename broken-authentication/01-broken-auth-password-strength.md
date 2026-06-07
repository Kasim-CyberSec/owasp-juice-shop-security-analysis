# Broken Authentication – Password Strength

**Category:** Broken Authentication  
**Difficulty:** ⭐⭐ (2 Star)  
**Severity:** 🟠 High  
**OWASP Top 10:** A07:2021 – Identification and Authentication Failures

---

## Description

The application allows login attempts against user accounts without implementing any rate limiting or account lockout mechanism. Combined with a weak administrator password, this makes the application vulnerable to brute-force attacks.

---

## Discovery Process

While browsing the application, I opened a product and expanded its reviews section. The review on the **Apple Juice (1000ml)** product was posted by `admin@juice-sh.op`, which revealed the administrator's email address.

> 📸 **Screenshot:** `screenshots/01-admin-email-exposed-review.png`

With the admin email identified, I navigated to the login page at `/#/login` and attempted several manual login attempts using common passwords. After approximately 10 attempts, I noticed:

- No account lockout was triggered
- No CAPTCHA appeared
- No delay or throttling was introduced
- Every failed attempt returned the same `401 Unauthorized` response: `"Invalid email or password."`

This confirmed that the endpoint had no brute-force protection.

---

## Exploitation

Since the endpoint had no rate limiting, I moved to **Burp Suite Intruder** to automate the attack.

**Steps:**

1. Intercepted the login POST request to `/rest/user/login`
2. Sent the request to **Intruder**
3. Set the `password` field as the payload position
4. Loaded **rockyou.txt** as the wordlist
5. Started the attack

**Request sent:**
```
POST /rest/user/login HTTP/1.1
Host: localhost:3000
Content-Type: application/json

{"email":"admin@juice-sh.op","password":"§password§"}
```

After **92 requests**, request #92 returned:

```
HTTP/1.1 200 OK
```

All other requests returned `401 Unauthorized` with length **413**.  
Request #92 returned `200 OK` with length **1185** — confirming a successful login.

**Valid credentials found:**
```
Email:    admin@juice-sh.op
Password: admin123
```

> 📸 **Screenshot:** `screenshots/02-burp-intruder-200-response.png`

---

## Impact

- Full administrative access to the application
- Access to all user data and orders
- Ability to modify or delete application content
- Potential for further exploitation of admin-only features

---

## Root Cause

Two separate issues combined to make this vulnerability exploitable:

1. **Weak password** — `admin123` is present in common wordlists such as rockyou.txt
2. **No brute-force protection** — the `/rest/user/login` endpoint applies no rate limiting, account lockout, or CAPTCHA

---

## Remediation

- Enforce a strong password policy (minimum length, complexity)
- Implement rate limiting per IP and per account on the login endpoint
- Lock the account temporarily after a defined number of failed attempts
- Introduce CAPTCHA after repeated failures
- Consider multi-factor authentication (MFA) for admin accounts

---

## Tools Used

Browser DevTools
Burp Suite Intruder
rockyou.txt

---

## References

- [OWASP – A07:2021 Identification and Authentication Failures](https://owasp.org/Top10/A07_2021-Identification_and_Authentication_Failures/)
- [OWASP Testing Guide – Testing for Weak Password Policy](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/04-Authentication_Testing/07-Testing_for_Weak_Password_Policy)
- [SecLists – rockyou.txt](https://github.com/danielmiessler/SecLists)
