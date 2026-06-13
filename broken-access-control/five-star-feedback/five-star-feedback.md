# Broken Access Control – Five-Star Feedback

**Category:** Broken Access Control
**Difficulty:** ⭐⭐ (2 Star)
**Severity:** 🟠 High
**OWASP Top 10:** A01:2021 – Broken Access Control

---

## Description

The feedback deletion endpoint (`DELETE /api/Feedbacks/:id`) allows any authenticated user to delete **any** feedback record by its numeric ID, regardless of whether the feedback belongs to them or whether they hold an administrative role. This makes it possible for a regular user to delete other users' feedback — including 5-star reviews — without any ownership or role check.

---

## Discovery Process

I logged in with a registered (non-admin) test account (`test@gmail.com`).
> 📸 **Screenshot:** `screenshots/01-test-account-registration.png`

I then submitted feedback through the Customer Feedback form, the same way as in the previous "Forged Feedback" challenge.
> 📸 **Screenshot:** `screenshots/02-feedback-form-filled.png`

Intercepting this submission in Burp Suite, the server returned `201 Created`, creating feedback record `id: 18` with `UserId: 23` — confirming my own account's ID and JWT for use in the next step.
> 📸 **Screenshot:** `screenshots/03-original-feedback-request.png`

To find a target, I sent a `GET /api/Feedbacks/` request, which returned the full list of feedback entries for **all users** — including their `id`, `UserId`, `comment`, and `rating` fields. Searching the response for `"rating":5`, I located a 5-star feedback entry (`id: 9`) that did not belong to my account.
> 📸 **Screenshot:** `screenshots/04-get-all-feedbacks-request.png`

While inspecting traffic in Burp Suite, I crafted a `DELETE /api/Feedbacks/9` request — `id: 9` being the 5-star feedback entry identified above — and sent it reusing the `Authorization: Bearer <JWT>` token from my own, non-admin session.
> 📸 **Screenshot:** `screenshots/05-delete-feedback-request.png`

The server processed the request successfully and deleted feedback `id: 9`, even though it belonged to another user and my account had no administrative privileges. The application confirmed the result directly.
> 📸 **Screenshot:** `screenshots/06-challenge-solved.png`

---

## Exploitation

**Steps:**

1. Authenticate as a normal (non-admin) user and capture the session's JWT from any request (`Authorization: Bearer <JWT>`)
2. Send `GET /api/Feedbacks/` to enumerate all feedback entries (across all users) and identify a 5-star record belonging to another user (`id: 9`)
3. Send a `DELETE /api/Feedbacks/9` request, reusing the attacker's own JWT in the `Authorization` header

**Request sent:**

```http
DELETE /api/Feedbacks/9 HTTP/1.1
Host: localhost:3000
Content-Length: 0
Authorization: Bearer <JWT_REDACTED>
Content-Type: application/json
Origin: http://localhost:3000
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: http://localhost:3000/
```

The server deleted the targeted feedback record without verifying that the requester owned `id: 9` or held an administrator role — confirming the challenge "Five-Star Feedback (Get rid of all 5-star customer feedback)" as solved.

---

## Impact

- Any authenticated user can permanently delete other users' feedback/reviews
- An attacker could systematically remove all positive (5-star) reviews from a product, manipulating its public reputation
- No ownership check or audit trail means deletions are silent and difficult to trace back to the responsible account

---

## Root Cause

The `DELETE /api/Feedbacks/:id` endpoint requires authentication (a valid JWT) but performs **no authorization check** — it never verifies that the `id` belongs to the requesting user, nor that the requesting user has an administrative role. This combines **Broken Object Level Authorization (BOLA)** with **missing function-level access control**: a destructive, admin-only operation is reachable by any logged-in user.

---

## Remediation

- On `DELETE /api/Feedbacks/:id`, verify that `feedback.UserId === req.user.id` **or** `req.user.role === 'admin'` before performing the deletion
- Apply role-based access control (RBAC) middleware to all admin-only routes, including feedback moderation endpoints
- Return `403 Forbidden` for unauthorized deletion attempts instead of silently succeeding
- Log all deletion operations together with the acting user's ID for auditability
- Add automated authorization tests covering "delete another user's resource" scenarios

---

## Tools Used

Browser DevTools
Burp Suite (Proxy & Repeater)

---

## References

- [OWASP Top 10 – A01:2021 Broken Access Control](https://owasp.org/Top10/A01_2021-Broken_Access_Control/)
- [OWASP API Security Top 10 – API1:2023 Broken Object Level Authorization](https://owasp.org/API-Security/editions/2023/en/0xa1-broken-object-level-authorization/)
- [OWASP API Security Top 10 – API5:2023 Broken Function Level Authorization](https://owasp.org/API-Security/editions/2023/en/0xa5-broken-function-level-authorization/)
