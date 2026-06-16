# Improper Input Validation – Admin Registration

**Category:** Improper Input Validation / Mass Assignment
**Difficulty:** ⭐⭐⭐ (3 Star)
**Severity:** 🟡 Medium
**OWASP Top 10:** A03:2021 – Injection (Mass Assignment) / A01:2021 – Broken Access Control

---

## Description

The user registration feature (`POST /api/Users/`) accepts a `role` field in the request body and stores it directly in the database without verifying or restricting it. As a result, any user registering an account can assign themselves the `admin` role simply by injecting `"role":"admin"` into the registration request

---

## Discovery Process

I registered a normal account through the registration form (`tester@gmail.com`) with a password, security question, and answer.
> 📸 **Screenshot:** `screenshots/01-user-registration-form.png`

Before submitting, I intercepted the request in Burp Suite. The JSON body contained the expected `email`, `password`, `passwordRepeat`, `securityQuestion`, and `securityAnswer` fields. I forwarded the request and the server returned `201 Created`, creating user record `id: 25` with `role: "customer"`.
> 📸 **Screenshot:** `screenshots/02-original-registration-request.png`

I noticed the `role` field was returned in the response body. This suggested the field might be writable from the client side — a classic sign of a mass assignment flaw. The only attacker-controlled value of interest was the `role` field injected into the POST body itself

---

## Exploitation

Since the endpoint never validated or restricted the `role` field, I used **Burp Suite Repeater** to resend the registration request with a modified body.

**Steps:**

1. Captured the `POST /api/Users/` registration request in Burp Suite
2. Sent the request to Repeater
3. Added `"role":"admin"` to the JSON body (using a new email, `test2@gmail.com`)
4. Forwarded the modified request

**Request sent:**

```http
POST /api/Users/ HTTP/1.1
Host: localhost:3000
Content-Type: application/json

{"email":"test2@gmail.com","password":"12345678","passwordRepeat":"12345678","role":"admin","securityQuestion":{"id":1,"question":"Your eldest siblings middle name?"},"securityAnswer":"12345678"}
```

**Response received:**

```http
HTTP/1.1 201 Created
Location: /api/Users/24

{"status":"success","data":{"username":"","deluxeToken":"","lastLoginIp":"0.0.0.0","profileImage":"/assets/public/images/uploads/defaultAdmin.png","isActive":true,"id":24,"email":"test2@gmail.com","role":"admin","deletedAt":null}}
```

> 📸 **Screenshot:** `screenshots/03-tampered-registration-request.png`

The server accepted the injected `role` field and created a new user record (`id: 24`) with `role: "admin"` — the admin profile image (`defaultAdmin.png`) further confirmed administrator privileges were granted.
> 📸 **Screenshot:** `screenshots/04-challenge-solved.png`

---

## Impact

- Any user can self-register an account with full administrator privileges, completely bypassing the privilege model
- Admin access exposes all user data, orders, and private application areas (confidentiality)
- An admin account can modify or delete any data in the application (integrity)
- Provides a persistent, attacker-controlled backdoor into the application (availability)

---

## Root Cause

The `POST /api/Users/` endpoint trusts the client-supplied `role` field and writes it directly to the database as the new user's role, instead of hardcoding the role server-side. This is a **Mass Assignment** flaw — a sensitive privilege field is exposed as a freely writable client input, typically caused by passing `req.body` directly into an ORM create call (e.g. `User.create(req.body)`)

---

## Remediation

- Always derive/hardcode the `role` value server-side (`role: "customer"`) on registration; never trust a `role` sent by the client
- Apply an allow-list of writable fields on the registration endpoint (`email`, `password`, `securityQuestion`, `securityAnswer` only)
- In Sequelize, restrict mass assignment with the `fields` option: `User.create(req.body, { fields: ['email','password','securityQuestion','securityAnswer'] })`
- Add automated tests that attempt to register with an elevated role and assert the created user is always a customer

---

## Tools Used

Browser DevTools
Burp Suite (Proxy & Repeater)

---

## References

- [OWASP Top 10 – A03:2021 Injection](https://owasp.org/Top10/A03_2021-Injection/)
- [OWASP Mass Assignment Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Mass_Assignment_Cheat_Sheet.html)
- [OWASP Top 10 – A01:2021 Broken Access Control](https://owasp.org/Top10/A01_2021-Broken_Access_Control/)
- [PortSwigger Web Security Academy – Access control vulnerabilities](https://portswigger.net/web-security/access-control)