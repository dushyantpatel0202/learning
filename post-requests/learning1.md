# Lesson 1: POST /api/auth/session (Firebase Session Exchange & Authentication)

In previous lessons, we mastered how clients send **GET** requests to retrieve data from the server.  
Now, we transition into **POST** requests ?" where the client sends a payload in the **HTTP Request Body** to create new resources, initialize sessions, and mutate state on the server.

Our first POST endpoint is the gateway to the entire authenticated ecosystem: **`POST /api/auth/session`**.  
This endpoint implements a secure **Token Exchange Pattern**: Firebase handles phone/email/password verification on the client, and our Express backend verifies Firebase's cryptographic proof, provisions a MongoDB user document, and issues our app's custom JWT.

---

## Roadmap of POST Requests
1. 🎯 **POST /api/auth/session** — Firebase token verification & JWT session creation (`AuthContext.jsx` & `authController.js`)
2. ⏳ **POST /api/doctors** — Creating a doctor profile & doctorCode generation (`DoctorDashboard.jsx` & `doctorController.js`)
3. ⏳ **POST /api/appointments** — Booking an appointment with consultation slot & fees (`BookingModal.jsx` & `appointmentController.js`)
4. ⏳ **POST /api/tokens/doctor/:doctorId** — Generating a live queue token for today (`TokenDisplay.jsx` & `tokenController.js`)
5. ⏳ **POST /api/uploads/signature** — Minting a secure Cloudinary upload signature (`PhotoUploader.jsx` & `uploadController.js`)

---

## Part 1 — Backend: Route Definition

File: [backend/routes/auth.js](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/backend/routes/auth.js)

```javascript
const express = require('express');
const asyncHandler = require('../middleware/asyncHandler');
const { createSession } = require('../controllers/authController');

const router = express.Router();

// Registering and logging in with a password now happen inside Firebase.
// This one endpoint is where Firebase's proof of identity becomes our app's login.
router.post('/session', asyncHandler(createSession));

module.exports = router;
```

### Line-by-Line Breakdown:
- `const router = express.Router();`: Initializes an isolated sub-router.
- `router.post('/session', ...)`: Registers an HTTP route that only responds to `POST` requests at `/api/auth/session` (since `app.js` mounts this file at `/api/auth`).
- `asyncHandler(createSession)`: Wraps the asynchronous controller to catch any unhandled promise rejections and hand them over to Express's global error handler.
- Notice that there is **no `protect` middleware** here. This is an open authentication handshake endpoint where unauthenticated clients present credentials to receive an authorization token.

---

## Part 2 — Backend: Controller Logic

File: [backend/controllers/authController.js](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/backend/controllers/authController.js)

```javascript
const jwt = require('jsonwebtoken');
const User = require('../models/User');
const { getFirebaseAuth, isConfigured } = require('../config/firebase');

function generateToken(user) {
  return jwt.sign(
    { id: user._id, role: user.role },
    process.env.JWT_SECRET,
    { expiresIn: '7d' }
  );
}

function publicUser(user) {
  return { id: user._id, name: user.name, email: user.email, phone: user.phone, role: user.role };
}

async function createSession(req, res) {
  const { idToken, name, role, phone } = req.body;

  if (!idToken || typeof idToken !== 'string') {
    return res.status(400).json({ message: 'Missing sign-in token.' });
  }
  if (!isConfigured()) {
    return res.status(503).json({
      message: 'Login is not set up on the server yet (missing Firebase settings).',
    });
  }

  // 1) Verify token with Firebase Admin SDK
  const firebaseAuth = getFirebaseAuth();
  let decoded;
  try {
    decoded = await firebaseAuth.verifyIdToken(idToken, true);
  } catch (err) {
    if (typeof err.code === 'string' && err.code.startsWith('auth/')) {
      return res.status(401).json({ message: 'Your sign-in could not be verified. Please sign in again.' });
    }
    throw err;
  }

  // 2) Enforce verification requirement (anti-spoofing)
  const hasVerifiedPhone = Boolean(decoded.phone_number);
  const hasVerifiedEmail = decoded.email_verified === true;
  if (!hasVerifiedPhone && !hasVerifiedEmail) {
    return res.status(403).json({
      code: 'not-verified',
      message: 'Please verify your email address first. Check your inbox for the link we sent.',
    });
  }

  // 3) Existing account lookup
  let user = await User.findOne({ firebaseUid: decoded.uid });
  let isNew = false;

  // 4) First-time user registration flow
  if (!user) {
    const cleanName = typeof name === 'string' ? name.trim() : '';
    if (!cleanName) {
      return res.json({
        needsProfile: true,
        email: decoded.email || null,
        phone: decoded.phone_number || null,
      });
    }
    if (cleanName.length < 2 || cleanName.length > 80) {
      return res.status(400).json({ message: 'Please enter your full name (2 to 80 characters).' });
    }

    let contactPhone = decoded.phone_number || '';
    if (!contactPhone && typeof phone === 'string' && phone.trim()) {
      const compact = phone.replace(/[\s-]/g, '');
      if (!/^\+?\d{8,15}$/.test(compact)) {
        return res.status(400).json({ message: 'Please enter a valid phone number, or leave it blank.' });
      }
      contactPhone = compact;
    }

    try {
      user = await User.create({
        firebaseUid: decoded.uid,
        name: cleanName,
        role: role === 'doctor' ? 'doctor' : 'patient',
        ...(decoded.email && { email: decoded.email.toLowerCase() }),
        ...(contactPhone && { phone: contactPhone }),
      });
      isNew = true;
    } catch (err) {
      if (err.code === 11000) user = await User.findOne({ firebaseUid: decoded.uid });
      if (!user) throw err;
    }
  }

  res.json({ token: generateToken(user), user: publicUser(user), isNew });
}
```

### Granular Line-by-Line Breakdown:
1. `const { idToken, name, role, phone } = req.body;`: Reads the parsed JSON payload from the request body. Unlike GET requests which use `req.query` or `req.params`, POST requests carry structured data inside `req.body`.
2. `if (!idToken || typeof idToken !== 'string')`: Defensive input validation. Rejects requests lacking a string token with `400 Bad Request`.
3. `await firebaseAuth.verifyIdToken(idToken, true)`: The parameter `true` enforces checkRevoked: it asks Firebase servers if the user session has been revoked or signed out remotely.
4. `if (typeof err.code === 'string' && err.code.startsWith('auth/'))`: Differentiates between invalid/expired client tokens (`401 Unauthorized`) and backend connection failures (`500 Internal Server Error`).
5. `const hasVerifiedPhone = Boolean(decoded.phone_number); const hasVerifiedEmail = decoded.email_verified === true;`: Critical security check preventing users with unverified email addresses from creating accounts or logging in.
6. `let user = await User.findOne({ firebaseUid: decoded.uid })`: Looks up the user in MongoDB by their immutable Firebase UID.
7. `if (!user) { ... if (!cleanName) return res.json({ needsProfile: true }); }`: If this is a first-time user and they haven't provided a profile name yet, the backend halts registration and asks the frontend to show the profile completion screen.
8. `if (err.code === 11000)`: Handles race conditions (e.g., rapid double-clicking of the submit button) where two concurrent requests try to insert the same `firebaseUid`.
9. `res.json({ token: generateToken(user), user: publicUser(user), isNew })`: Signs our own app JWT (7-day validity containing `_id` and `role`) and strips sensitive database fields using `publicUser`.

---

## Part 3 — Frontend: The Auth Provider & Service Layer

File: [frontend/src/context/AuthContext.jsx](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/frontend/src/context/AuthContext.jsx)

```javascript
// Step 2 of login: Firebase signed the user in; exchange for server token
async function completeSignIn(profile = {}) {
  const firebaseUser = auth?.currentUser;
  if (!firebaseUser) throw new Error('Please sign in first.');

  // "true" = force refresh Firebase token to pick up recent email verification
  const idToken = await firebaseUser.getIdToken(true);

  let pending = {};
  try {
    pending = JSON.parse(sessionStorage.getItem('pendingProfile')) || {};
  } catch {
    pending = {};
  }

  const { data } = await api.post('/auth/session', { idToken, ...pending, ...profile });
  if (data.needsProfile) return data;

  sessionStorage.removeItem('pendingProfile');
  localStorage.setItem('token', data.token);
  localStorage.setItem('user', JSON.stringify(data.user));
  setUser(data.user);
  return data;
}
```

### Line-by-Line Breakdown:
- `const idToken = await firebaseUser.getIdToken(true);`: Retrieves Firebase's cryptographically signed JWT. Passing `true` forces a token refresh from Firebase, which is essential if the user just verified their email in another tab.
- `const { data } = await api.post('/auth/session', { ... })`: Dispatches the HTTP POST request to Axios with `{ idToken, name, role, phone }` formatted as a JSON body.
- `localStorage.setItem('token', data.token);`: Persists our backend JWT in the browser's `localStorage`. The Axios interceptor in [api.js](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/frontend/src/utils/api.js) automatically reads this token and injects it into the `Authorization: Bearer <token>` header on subsequent requests.
- `setUser(data.user);`: Updates global React state, causing the navigation bar, protected routes, and dashboards to immediately reflect the logged-in user.

---

## Part 4 — Frontend: First-Time Profile Completion Form

File: [frontend/src/pages/CompleteProfile.jsx](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/frontend/src/pages/CompleteProfile.jsx)

```javascript
export default function CompleteProfile() {
  const { finishSignIn } = useAuth();
  const [name, setName] = useState('');
  const [role, setRole] = useState('patient');
  const [phone, setPhone] = useState('');
  const [busy, setBusy] = useState(false);

  async function handleSubmit(e) {
    e.preventDefault();
    setBusy(true);
    try {
      await finishSignIn({ name, role, ...(phone.trim() && { phone }) });
    } catch (err) {
      setError(authErrorMessage(err));
    } finally {
      setBusy(false);
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      <Field label="Full name" value={name} onChange={(e) => setName(e.target.value)} required />
      <input type="radio" checked={role === 'patient'} onChange={() => setRole('patient')} />
      <input type="radio" checked={role === 'doctor'} onChange={() => setRole('doctor')} />
      <button type="submit" disabled={busy}>{busy ? 'Saving...' : 'Continue'}</button>
    </form>
  );
}
```

### Key UI Principles:
- **Prevent Default**: `e.preventDefault()` halts standard browser form submission to preserve single-page application state.
- **Controlled Components**: Form inputs are driven by React state (`name`, `role`, `phone`), ensuring sanitized payloads.
- **Button Disable State (`busy`)**: Prevents duplicate POST calls while the server verifies and creates the account.

---

## Part 5 — The Full End-to-End Data Flow

```
[Browser / User]
       │
       ▼ 1. User signs in via Firebase Auth (Email/Password or Phone OTP)
[Firebase Client SDK] -> Returns authenticated `firebaseUser`
       │
       ▼ 2. getIdToken(true) generates fresh Firebase ID Token
[AuthContext.jsx]
       │
       ▼ 3. HTTP POST /api/auth/session
            Headers: Content-Type: application/json
            Body: { idToken: "ey...", name: "Dr. Aryan", role: "doctor" }
       │
       ▼ 4. Express app.js parses body via express.json()
[routes/auth.js]
       │
       ▼ 5. Passes to createSession in authController.js
[authController.js]
       │
       ├─► 6. firebaseAuth.verifyIdToken(idToken, true) -> Confirms authenticity
       ├─► 7. Validates decoded.email_verified || decoded.phone_number
       ├─► 8. Queries User.findOne({ firebaseUid }) in MongoDB
       ├─► 9. If new user -> User.create({ firebaseUid, name, role, ... })
       └─► 10. Generates backend JWT with user._id & role
       │
       ▼ 11. Backend Responds:
            HTTP 200 OK
            Body: { token: "app_jwt_xyz", user: { ... }, isNew: true }
       │
       ▼ 12. AuthContext stores token in localStorage and updates React state
[React App] -> User redirected to /doctor-dashboard or / based on role!
```

---

## Part 6 — Quick Self-Test

1. **Why does the client send Firebase's `idToken` to our backend instead of just sending `{ email, password }`?**  
   *Because our server never touches or stores raw user passwords.* Firebase handles hashing, rate-limiting, and Multi-Factor Authentication. Our backend only verifies the cryptographic signature on Firebase's `idToken`.

2. **Why does `POST /api/auth/session` return `{ needsProfile: true }` instead of failing if `name` is missing?**  
   *To enable flexible multi-step onboarding.* Third-party authentications (like phone OTP or email links) do not provide a name upfront. Returning `{ needsProfile: true }` instructs the client to display the `<CompleteProfile />` view without dropping the authenticated Firebase session.

3. **What is the difference between how data is sent in a GET request versus this POST request?**  
   *GET requests send parameters via URL query strings or path parameters (e.g. `/api/doctors?search=cardio`), visible in server access logs and browser history.* POST requests transmit structured objects inside the HTTP request body (`req.body`) wrapped in `application/json`, which supports large nested payloads and prevents sensitive tokens from leaking into server URLs.
