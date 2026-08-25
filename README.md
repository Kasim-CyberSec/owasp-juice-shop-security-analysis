# OWASP Juice Shop Security Assessment

## Overview

This repository documents hands-on web application security testing and vulnerability writeups completed against OWASP Juice Shop, an intentionally vulnerable application designed for security training.

The purpose of this project is to practice identifying, validating, and documenting common web application security vulnerabilities in an authorized lab environment.

Each finding documents the testing process, technical evidence, security impact, root cause, and remediation recommendations.

---

## Scope

The testing documented in this repository was performed against a local OWASP Juice Shop instance in an authorized lab environment.

The currently documented areas include:

* Broken Access Control
* Authentication Security
* Server-Side Input Validation
* Authorization Testing

---

## Testing Approach

The general testing approach used throughout the project included:

1. Exploring the targeted application functionality.
2. Observing normal application behavior and HTTP requests.
3. Intercepting relevant requests with Burp Suite.
4. Identifying user-controlled parameters and security-sensitive functionality.
5. Modifying and replaying requests to test application security controls.
6. Validating identified vulnerabilities manually.
7. Collecting technical evidence and screenshots.
8. Documenting the impact, root cause, and recommended remediation.

---

## Lab Environment

* OWASP Juice Shop
* Local authorized testing environment

---

## Tools Used

* Burp Suite

  * Proxy
  * Repeater
  * Intruder
* Web Browser
* rockyou.txt for password strength testing

---

# Documented Findings

## 1. Broken Access Control

Testing focused on whether an authenticated user could access or modify resources without proper authorization.

### Five-Star Feedback

An authenticated non-admin user was able to delete feedback belonging to another user because the application did not properly enforce authorization before processing the request.

➡️ [View detailed writeup](./broken-access-control/five-star-feedback/five-star-feedback.md)

### Forged Feedback

The feedback creation request contained a client-controlled `UserId` value. Modifying this value allowed feedback to be submitted as another user.

➡️ [View detailed writeup](./broken-access-control/forged-feedback/forged-feedback.md)

---

## 2. Authentication Security

### Password Strength / Brute-Force Protection

The login functionality was tested for resistance to repeated authentication attempts.

After manual testing, Burp Suite Intruder was used to automate password attempts against the login endpoint. The testing demonstrated insufficient protection against repeated login attempts.

➡️ [View detailed writeup](./broken-authentication/broken-auth-password-strength.md)

---

## 3. Improper Input Validation

Testing focused on whether security-sensitive input was properly validated by the server rather than relying only on client-side restrictions.

### Admin Registration

The registration request was intercepted and modified to include a client-controlled `role` parameter.

The server accepted the modified request and created a user with administrative privileges, demonstrating insufficient server-side validation of security-sensitive registration parameters.

➡️ [View detailed writeup](./improper-input-validation/admin-registration/admin-registration.md)

### Empty User Registration

The registration request was modified to submit empty values for required account fields.

The server accepted the request despite the invalid input, demonstrating missing or insufficient server-side validation.

➡️ [View detailed writeup](./improper-input-validation/empty-user-registration/empty-user-registration.md)

---

## Security Lessons

The documented findings highlight several important secure development principles:

* Authorization must always be enforced on the server side.
* Applications should not trust client-controlled ownership or privilege-related parameters.
* User identity should be derived from the authenticated session whenever possible.
* Security-sensitive parameters should be validated using strict server-side controls.
* Required input validation should not rely only on frontend restrictions.
* Authentication endpoints should include protections against repeated automated login attempts.
* Security controls should be validated independently of the client application.

Detailed remediation recommendations are included in the individual vulnerability writeups.

---

## Skills Demonstrated

This project demonstrates practical experience with:

* Web application security testing
* HTTP request and response analysis
* Burp Suite Proxy
* Burp Suite Repeater
* Burp Suite Intruder
* Authentication testing
* Authorization testing
* Broken Access Control testing
* Server-side input validation testing
* HTTP request manipulation
* Manual vulnerability validation
* Evidence collection
* Root cause analysis
* Security impact analysis
* Vulnerability documentation
* Remediation recommendations

---

## Repository Structure

```text
owasp-juice-shop-security-analysis/
│
├── broken-access-control/
│   ├── five-star-feedback/
│   │   └── five-star-feedback.md
│   │
│   └── forged-feedback/
│       └── forged-feedback.md
│
├── broken-authentication/
│   └── broken-auth-password-strength.md
│
├── improper-input-validation/
│   ├── admin-registration/
│   │   └── admin-registration.md
│   │
│   └── empty-user-registration/
│       └── empty-user-registration.md
│
└── README.md
```

Each finding contains a separate technical writeup with supporting evidence where applicable.

---

## Disclaimer

This project was created for educational and cybersecurity training purposes.

All testing documented in this repository was performed against OWASP Juice Shop in an authorized local lab environment. OWASP Juice Shop is intentionally vulnerable and designed specifically for security training.

No testing was performed against unauthorized systems.
