# Module 11: Frontend Security & Authentication Architecture

---

## 1. 💡 Mental Model: Client-Side Security Realities

The golden rule of Full-Stack Web Development:
> [!CAUTION]
> **Never trust the client.**
> Any JavaScript running in a user's browser can be inspected, paused, modified, or bypassed via browser DevTools. Frontend route guards are for **User Experience (UX)**; backend authorization middleware is for **Security**.

---

## 2. 🛡️ Defending Against XSS (Cross-Site Scripting)

**XSS** occurs when an attacker injects malicious JavaScript into your application that runs in other users' browsers (stealing auth tokens or hijacking accounts).

### How React Protects You By Default:
React automatically escapes all strings rendered inside JSX brackets `{}`:
```jsx
// Attacker enters this as their name:
const attackerName = "<script>stealTokens()</script>";

// React safely encodes this to: &lt;script&gt;stealTokens()&lt;/script&gt;
<h1>Welcome, {attackerName}</h1>
```

### The Dangerous Vulnerabilities in React:
#### 1. `dangerouslySetInnerHTML`
If you render unsanitized user content using `dangerouslySetInnerHTML`:
```jsx
// ❌ CRITICAL VULNERABILITY:
<div dangerouslySetInnerHTML={{ __html: doctor.bio }} />

// ✅ SAFE: Sanitize with DOMPurify first:
import DOMPurify from 'dompurify';
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(doctor.bio) }} />
```

#### 2. The `javascript:` URL Injection Vulnerability
```jsx
// If doctor submits: doctor.website = "javascript:stealCredentials()"
<a href={doctor.website}>Visit Doctor's Website</a>

// ✅ SAFE: Validate protocol before rendering:
function isSafeUrl(url) {
  return url.startsWith('http://') || url.startsWith('https://');
}
<a href={isSafeUrl(doctor.website) ? doctor.website : '#'}>Visit Website</a>
```

---

## 3. 🔐 The Great Debate: `localStorage` vs `httpOnly` Cookies

Every MERN stack interview asks this question:
**"Where should you store JWT tokens in the browser?"**

| Strategy | Where it's stored | Vulnerable to XSS? | Vulnerable to CSRF? | How to implement |
| :--- | :--- | :--- | :--- | :--- |
| **`localStorage`** | Browser Storage API | **YES** (Any malicious JS can read `localStorage.getItem('token')`) | **NO** (Not automatically attached to requests) | Standard in SPAs with Authorization headers |
| **`httpOnly` Cookie** | Browser Cookie Jar | **NO** (JavaScript cannot read `document.cookie` if `httpOnly` flag is set) | **YES** (Browser sends cookie automatically to origin) | Requires `SameSite: 'strict'` or CSRF tokens |

### The Production Best-Practice Architecture:
1. **Short-lived Access Token (JWT)**: Kept in React memory (or `localStorage` with strict CSP). Expires in 15 minutes.
2. **Long-lived Refresh Token**: Stored in a secure `httpOnly`, `SameSite=Strict`, `Secure` cookie. Used only to refresh the access token.

---

## 4. 🏥 Real Project Authentication Flow: Firebase $\leftrightarrow$ Node.js Token Bridge

In `frontend/src/context/AuthContext.jsx`, we implement a secure two-tier authentication bridge:

```
[User signs in with OTP / Email]
           │
           ▼
[Firebase Auth Client] ─── (Generates cryptographically signed idToken)
           │
           ▼
[POST /api/auth/session] ── (Passes Firebase idToken to our Express server)
           │
           ▼
[Express Backend] ────── (Verifies token signature using `firebase-admin` SDK)
           │
           ▼
[Generates App JWT] ──── (Returns custom session token with role: 'patient' | 'doctor')
```

### Why this design is secure:
1. The frontend never decides its own user role. Even if a patient modifies `localStorage.setItem('user', JSON.stringify({ role: 'doctor' }))`, our Express backend independently checks the cryptographic signature of the JWT before allowing access to doctor records!

---

## 5. 🎯 Interview Question Bank: Frontend Security

### Q1: Can a user bypass a frontend `ProtectedRoute`?
> **Answer**:
> Yes. A user can open DevTools, edit React component state, modify `localStorage`, or inspect source code.
> That is why frontend route protection is purely for **navigation experience** (avoiding ugly UI errors). The **true security boundary is the backend API**, which must verify the JWT token and user permissions on every single protected endpoint (`/api/doctor/dashboard`, `/api/appointments/cancel`).

### Q2: What is Content Security Policy (CSP)?
> **Answer**:
> An HTTP response header (or `<meta>` tag) that restricts where scripts, styles, images, and fonts can be loaded from.
> For example:
> `Content-Security-Policy: default-src 'self'; script-src 'self' https://apis.google.com;`
> prevents attackers from executing injected scripts loaded from unauthorized external domains.

### Q3: What is Clickjacking and how do you prevent it?
> **Answer**:
> Clickjacking is an attack where an attacker embeds your website inside a transparent `<iframe>` on their malicious site, tricking users into clicking buttons (like "Confirm Payment" or "Delete Account").
> Prevented by setting the HTTP header `X-Frame-Options: DENY` or CSP `frame-ancestors 'none'`.

---

## 6. 💻 Hands-On Mastery Exercise

Look at `DoctorProfile.jsx` where `mapsUrl` is constructed. Write a sanitization utility function to ensure external URLs are strictly validated before rendering links.
