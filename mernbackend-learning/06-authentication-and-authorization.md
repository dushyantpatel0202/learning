# 06. Authentication and authorization

Authentication establishes who the caller is. Authorization decides whether that caller may perform a specific action on a specific resource. A valid login alone does not grant access to every record.

## Request flow

1. The client obtains a credential through a trusted sign-in flow.
2. The server verifies the credential, including signature, issuer, audience, and expiry when applicable.
3. Authentication middleware attaches a trusted user identity to the request.
4. Route logic checks roles and resource ownership before reading or changing protected data.

Never trust a user ID, role, or ownership claim supplied in the request body. Scope database queries to the authenticated identity where possible, such as finding an appointment by both its ID and the current patient's ID.

## Sessions and tokens

Cookie sessions and bearer tokens have different storage and revocation tradeoffs. Use established libraries and secure defaults. For cookies, configure `HttpOnly`, `Secure` in HTTPS deployments, and an appropriate `SameSite` policy; protect state-changing requests against CSRF when the cookie setup requires it. For bearer tokens, avoid storing long-lived credentials in browser-accessible storage when a safer design is available.

Passwords should be hashed with a password-hashing algorithm designed for passwords, using a maintained library. Never store plaintext passwords or invent your own hashing scheme.

## Practice

Write an authorization rule for a patient who can read or cancel only their own appointment, and a doctor who can view appointments assigned to them. Test both allowed and denied cases.
