# Improper Input Validation – Empty User Registration

**Category:** Improper Input Validation  
**Difficulty:** ⭐ (1 Star)  
**Severity:** 🟠 Low  
**OWASP Top 10:** A04:2021 – Insecure Design

---

## Description

The application fails to perform server-side validation on the registration endpoint. While the frontend enforces input requirements through Angular form validators, these controls can be bypassed by intercepting and modifying the HTTP request using a proxy tool like Burp Suite. This creates a validation gap where the backend accepts empty strings for critical fields like email and password.

---

## Discovery Process

### Step 1 – Fill the Registration Form

I navigated to `http://localhost:3000/#/register` and filled the form with test data:

- **Email:** `test@gmail.com`
- **Password:** `12345678`
- **Repeat Password:** `12345678`
- **Security Question:** Your eldest siblings middle name?
- **Answer:** `12345678`

> 📸 **Screenshot 1:** `screenshot/01-registration-form-filled.png`

---

### Step 2 – Intercept the Request with Burp Suite

I enabled Intercept in Burp Suite and clicked Register. Burp captured the outgoing `POST /api/Users/` request.

> 📸 **Screenshot 2:** `screenshot/02-burp-original-request.png`

---

### Step 3 – Modify the Request

I changed `email`, `password`, and `passwordRepeat` to empty strings:

```json
{
  "email": "",
  "password": "",
  "passwordRepeat": "",
  "securityQuestion": {...},
  "securityAnswer": "12345678"
}
```

> 📸 **Screenshot 3:** `screenshot/03-burp-modified-request-empty-fields.png`

---

### Step 4 – Forward the Request

I clicked Forward. The server responded with **`201 Created`** — the account was registered with no validation error.

> 📸 **Screenshot 4:** `screenshot/04-challenge-solved-banner.png`

---

## Technical Analysis

The vulnerability exists due to a validation gap between the frontend and the backend:

| Layer | Validation Present? | Notes |
|-------|---------------------|-------|
| Frontend | ✅ Yes | Angular validators + HTML5 `required` attributes |
| Backend API | ❌ No | `POST /api/Users/` accepts empty strings directly |

The Angular form correctly prevents submission with empty fields. However, the backend creates the account without verifying that the received fields are non-empty — it blindly trusts whatever data arrives in the request body.

---

## Impact Assessment

| CIA Dimension | Impact | Explanation |
|---------------|--------|-------------|
| Confidentiality | Low | A ghost account with empty credentials could be used to probe the system anonymously without being traced. |
| Integrity | Low | The database accepts and stores invalid records with empty critical fields. |
| Availability | Low | The lack of validation could be scripted to flood the database with empty accounts, degrading performance. |

---

## Root Cause Analysis

The development team trusted the client — they assumed the Angular validators were sufficient to enforce input rules. The backend `POST /api/Users/` endpoint was never hardened with its own validation logic. The developer never considered that an attacker could intercept and modify the request before it reaches the server.

---

## Remediation

Server-side validation must be enforced on the `POST /api/Users/` endpoint:

```javascript
const { body, validationResult } = require('express-validator');

app.post('/api/Users/', [
  body('email').isEmail().normalizeEmail(),
  body('password').isLength({ min: 8 }).notEmpty(),
  body('passwordRepeat').notEmpty()
], (req, res) => {
  const errors = validationResult(req);
  if (!errors.isEmpty()) {
    return res.status(400).json({ errors: errors.array() });
  }
  // proceed with registration
});
```

**Best Practices:**

1. Never trust client-side validation alone — always re-validate on the server.
2. Use a dedicated validation library like `express-validator` or `Joi`.
3. Return **400 Bad Request** for invalid inputs.
4. Apply strict schema validation on all incoming API request bodies.

---

## Lessons Learned

- Never trust user input — any data coming from the client can be modified before reaching the server.
- Frontend validation is a UX feature, not a security control.
- The backend is the last line of defense — it must validate all inputs independently.

---

## Real-World Relevance

This vulnerability appears in any application with a user registration flow. It is particularly common in:

- **Early-stage startups** — teams prioritize shipping the product and defer security hardening.
- **Legacy applications** — APIs built before security-first practices were adopted.
- **Mobile apps** — developers assume only their app will call the backend API.

In real-world bug bounty programs, this has led to mass account creation for spam and fraud, account enumeration, and authentication bypass when combined with other vulnerabilities.

---

## Tools Used

- **Burp Suite** — HTTP proxy for intercepting and modifying requests
- **Browser (Chrome)** — for interacting with the Juice Shop UI

---

## References

- [OWASP Input Validation Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Input_Validation_Cheat_Sheet.html)
- [CWE-20: Improper Input Validation](https://cwe.mitre.org/data/definitions/20.html)
- [OWASP A04:2021 – Insecure Design](https://owasp.org/Top10/A04_2021-Insecure_Design/)
- [express-validator Documentation](https://express-validator.github.io/docs/)
