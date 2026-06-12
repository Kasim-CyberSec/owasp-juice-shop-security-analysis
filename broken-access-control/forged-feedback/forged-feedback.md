# Broken Access Control – Forged Feedback

**Category:** Broken Access Control
**Difficulty:** ⭐⭐⭐ (3 Star)
**Severity:** 🟡 Medium
**OWASP Top 10:** A01:2021 – Broken Access Control

---

## Description

The "Customer Feedback" feature (`POST /api/Feedbacks`) accepts a `UserId` field in the request body and stores it directly in the database without verifying that it matches the ID of the currently authenticated user. As a result, any logged-in user can submit feedback that is permanently attributed to a different user's account

---

## Discovery Process

I logged in with a registered test account (`test@gmail.com`).
> 📸 **Screenshot:** `screenshots/01-test-account-registration.png`

I opened the **Customer Feedback** form, entered a comment, set a rating, and solved the CAPTCHA.
> 📸 **Screenshot:** `screenshots/02-feedback-form-filled.png`

Before submitting, I intercepted the request in Burp Suite. The JSON body contained the expected `comment`, `rating`, `captchaId`, and `captcha` fields — but also a `UserId` field set to my own account's ID (`23`). I forwarded the request and the server returned `201 Created`, creating feedback record `id: 18` with `UserId: 23`.
> 📸 **Screenshot:** `screenshots/03-original-feedback-request.png`

I then noted another user's numeric ID (`1`) from the feedback list and tried a few alternative HTTP methods (GET, PUT, PATCH) on the endpoint — none of them changed the behavior. The only attacker-controlled value of interest was the `UserId` field in the POST body itself

---

## Exploitation

Since the endpoint never checked `UserId` against the session, I used **Burp Suite Repeater** to resend the request with a modified body.

**Steps:**

1. Captured the `POST /api/Feedbacks` request while logged in as the attacker account (`UserId: 23`)
2. Sent the request to Repeater
3. Changed `"UserId": 23` to `"UserId": 1` (the victim's account ID)
4. Forwarded the modified request

**Request sent:**

```http
POST /api/Feedbacks HTTP/1.1
Host: localhost:3000
Content-Type: application/json

{"UserId":1,"captchaId":4,"captcha":"66","comment":"helloooooooooooooooooooo (***t@gmail.com)","rating":2}
```

**Response received:**

```http
HTTP/1.1 201 Created
Location: /api/Feedbacks/19

{"status":"success","data":{"id":19,"UserId":1,"comment":"helloooooooooooooooooooo (***t@gmail.com)","rating":2}}
```

> 📸 **Screenshot:** `screenshots/04-tampered-feedback-request.png`

The server accepted the forged `UserId` and created a new feedback record (`id: 19`) attributed to user `1` — confirming feedback could be posted in another user's name.
> 📸 **Screenshot:** `screenshots/05-challenge-solved.png`

---

## Impact

- Ability to create feedback/content permanently attributed to another user's account
- Could be used to post offensive, defamatory, or misleading content "in someone else's name"
- Confirms internal numeric user IDs are valid, writable identifiers — useful for further IDOR/enumeration attacks

---

## Root Cause

The `POST /api/Feedbacks` endpoint trusts the client-supplied `UserId` field and writes it directly to the database as the feedback's owner, instead of deriving ownership from the authenticated session/JWT. This is a **Broken Object Level Authorization (BOLA) / IDOR** flaw combined with mass assignment — an internal foreign-key field is exposed as a freely writable client input

---

## Remediation

- Always derive `UserId` for new feedback records from the authenticated session/JWT (`req.user.id`); never trust a `UserId` sent by the client
- Apply an allow-list of writable fields on the feedback creation endpoint (`comment`, `rating`, `captchaId`, `captcha` only)
- Add object-level authorization checks on all create/update/delete endpoints, not just read endpoints
- Add automated tests that attempt to set ownership fields to other users' IDs and assert they are rejected/ignored

---

## Tools Used

Browser DevTools
Burp Suite (Proxy & Repeater)

---

## References

- [OWASP API Security Top 10 – API1:2023 Broken Object Level Authorization](https://owasp.org/API-Security/editions/2023/en/0xa1-broken-object-level-authorization/)
- [OWASP Top 10 – A01:2021 Broken Access Control](https://owasp.org/Top10/A01_2021-Broken_Access_Control/)
- [PortSwigger Web Security Academy – Access control vulnerabilities](https://portswigger.net/web-security/access-control)