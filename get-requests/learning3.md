# Lesson 3: GET /api/tokens/doctor/:doctorId (Live Token & Queue Status)

In **Lesson 2**, we learned how clicking a doctor's card requests `GET /api/doctors/:id` to load their full profile into `DoctorProfile.jsx`.  
Inside that profile page, patients need to know in real time:
- *Is the doctor currently seeing patients?*
- *Which token number is inside the consultation room right now?*
- *How many patients are waiting ahead of me?*

In **Lesson 3**, we dissect **`GET /api/tokens/doctor/:doctorId`** — the endpoint powering the live token widget inside [TokenDisplay.jsx](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/frontend/src/components/TokenDisplay.jsx). We will also learn how client-side polling with `setInterval` keeps the queue fresh without requiring heavy WebSocket servers.

---

## Roadmap of GET Requests
1. ✅ **GET /api/doctors** — Fetching doctors list with search and filters (`Home.jsx`)
2. ✅ **GET /api/doctors/:id** — Fetching a single doctor's complete profile (`DoctorProfile.jsx`)
3. 🎯 **GET /api/tokens/doctor/:doctorId** — Live token and queue status for a doctor (`TokenDisplay.jsx`)
4. ⏳ **GET /api/appointments/my** & **GET /api/tokens/my** — Patient's appointments and tokens (`PatientAppointments.jsx`)
5. ⏳ **GET /api/doctors/me/profile** — Logged-in doctor's profile (`DoctorDashboard.jsx`)
6. ⏳ **GET /api/tokens/doctor/:doctorId/queue** & **GET /api/appointments/doctor** — Doctor's live patient queue & appointments list (`DoctorDashboard.jsx`)

---

## Part 1 — Backend: Route Definition

### 1. Root Mount in `backend/app.js`

**File:** [app.js](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/backend/app.js#L26)

```javascript
const tokenRoutes = require('./routes/tokens');
// ...
app.use('/api/tokens', tokenRoutes);
```

Any route defined in `routes/tokens.js` is prefixed with `/api/tokens`.

---

### 2. Route Declaration in `backend/routes/tokens.js`

**File:** [tokens.js](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/backend/routes/tokens.js#L14-L18)

```javascript
const express = require('express');
const asyncHandler = require('../middleware/asyncHandler');
const { protect, doctorOnly } = require('../middleware/auth');
const {
  getTokenStatus,
  generateToken,
  callNextPatient,
  getDoctorQueue,
  getMyTokens,
} = require('../controllers/tokenController');

const router = express.Router();

router.get('/my', protect, asyncHandler(getMyTokens));
router.get('/doctor/:doctorId/queue', protect, doctorOnly, asyncHandler(getDoctorQueue));
router.get('/doctor/:doctorId', asyncHandler(getTokenStatus));
router.post('/doctor/:doctorId', protect, asyncHandler(generateToken));
router.put('/doctor/:doctorId/next', protect, doctorOnly, asyncHandler(callNextPatient));

module.exports = router;
```

### Line-by-Line Breakdown:

- **`router.get('/doctor/:doctorId', asyncHandler(getTokenStatus));`**:
  - Registers an HTTP `GET` route on the sub-path `/doctor/:doctorId`.
  - Together with the app-level mount (`/api/tokens`), the full URL becomes:  
    `GET /api/tokens/doctor/:doctorId`
  - `:doctorId` is a route parameter. When a request like `GET /api/tokens/doctor/64f8a12bc8...` hits Express, Express places `"64f8a12bc8..."` into `req.params.doctorId`.
- **Public Access (No `protect` middleware)**:
  - Notice that unlike `/my` or `/doctor/:doctorId/queue`, this route does **not** have `protect`.
  - Anyone visiting a doctor's profile (whether logged in, registered, or a casual guest) has the right to see the live queue count and current token number.
- **Route Ordering Nuance**:
  - `router.get('/doctor/:doctorId/queue', ...)` appears **before** `router.get('/doctor/:doctorId', ...)`.
  - If a route with an extra segment (`/doctor/:doctorId/queue`) were defined after a generic dynamic pattern, conflicts could occur depending on Express wildcard configurations. Keeping more specific sub-paths higher is always best practice.

---

## Part 2 — Backend: Controller & Database Models

### 1. Database Schema Context

Let's look at the two Mongoose models collaborating here:

#### `Doctor` Model ([Doctor.js](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/backend/models/Doctor.js#L54-L58)):
```javascript
tokenSystem: {
  nowServing: { type: Number, default: 0 },
  queueLength: { type: Number, default: 0 },
  avgWaitMinutes: { type: Number, default: 10 },
}
```
- `nowServing`: The token number currently inside the consultation room.
- `avgWaitMinutes`: The doctor's estimated time per consultation (e.g. 10 minutes).

#### `Token` Model ([Token.js](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/backend/models/Token.js#L7-L20)):
```javascript
const tokenSchema = new mongoose.Schema(
  {
    doctor: { type: mongoose.Schema.Types.ObjectId, ref: 'Doctor', required: true },
    patient: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
    tokenNumber: { type: Number, required: true },
    date: { type: String, required: true }, // "YYYY-MM-DD"
    status: {
      type: String,
      enum: ['waiting', 'serving', 'done', 'cancelled'],
      default: 'waiting',
    },
  },
  { timestamps: true }
);
```

---

### 2. Controller Logic in `backend/controllers/tokenController.js`

**File:** [tokenController.js](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/backend/controllers/tokenController.js#L4-L24)

```javascript
const Doctor = require('../models/Doctor');
const Token = require('../models/Token');

function todayString() {
  return new Date().toISOString().slice(0, 10);
}

// GET /api/tokens/doctor/:doctorId
async function getTokenStatus(req, res) {
  const doctor = await Doctor.findById(req.params.doctorId);
  if (!doctor) return res.status(404).json({ message: 'Doctor not found.' });

  const waitingCount = await Token.countDocuments({
    doctor: doctor._id,
    date: todayString(),
    status: 'waiting',
  });

  res.json({
    nowServing: doctor.tokenSystem.nowServing,
    queueLength: waitingCount,
    avgWaitMinutes: doctor.tokenSystem.avgWaitMinutes,
  });
}
```

### Line-by-Line Breakdown:

- **`function todayString()`**:
  - `new Date().toISOString()` returns an ISO string like `"2026-10-20T08:30:00.000Z"`.
  - `.slice(0, 10)` extracts `"2026-10-20"` (`YYYY-MM-DD`).
  - This guarantees token queues reset automatically every single calendar day! Yesterday's tokens will never bleed into today's queue.
- **`const doctor = await Doctor.findById(req.params.doctorId);`**:
  - Queries MongoDB's `doctors` collection using the ID from the route parameter.
  - If no doctor matches, returns `404 Not Found`.
- **`const waitingCount = await Token.countDocuments(...)`**:
  - Instead of fetching every token document into Node.js memory and counting them in JavaScript, Mongoose executes an optimized MongoDB `$count` query:
    1. `doctor: doctor._id`: Only tokens for this specific doctor.
    2. `date: todayString()`: Only tokens issued today.
    3. `status: 'waiting'`: Only patients who haven't been called in yet (`status` is not `'serving'`, `'done'`, or `'cancelled'`).
  - This is extremely fast even with thousands of records.
- **`res.json({ nowServing, queueLength, avgWaitMinutes })`**:
  - Returns a clean, lightweight JSON response:
    ```json
    {
      "nowServing": 4,
      "queueLength": 3,
      "avgWaitMinutes": 10
    }
    ```

---

## Part 3 — Frontend: Where `TokenDisplay` is Mounted

In `frontend/src/pages/DoctorProfile.jsx`:

```jsx
// Line 169 in DoctorProfile.jsx
<TokenDisplay doctorId={doctor._id} className="mt-0" />
```

As soon as `DoctorProfile` finishes loading the doctor object, it passes the doctor's database `_id` into `<TokenDisplay />` as a prop.

---

## Part 4 — Frontend: Component Logic (`TokenDisplay.jsx`)

**File:** [TokenDisplay.jsx](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/frontend/src/components/TokenDisplay.jsx)

### 1. State Declarations

```jsx
export default function TokenDisplay({ doctorId, className }) {
  const [status, setStatus] = useState(null);
  const [myToken, setMyToken] = useState(null);
  const [error, setError] = useState('');
  const { user } = useAuth();
  const navigate = useNavigate();
```

| State / Hook | Initial Value | Purpose |
|---|---|---|
| `status` | `null` | Stores `{ nowServing, queueLength, avgWaitMinutes }` returned from the backend. |
| `myToken` | `null` | Stores the logged-in user's token object if they click "Get Token". |
| `error` | `''` | Stores error messages if a token request fails. |
| `user` | Object / `null` | Checked when the user clicks "Get Token" to ensure they are authenticated. |

---

### 2. Polling with `useEffect` and Cleanup

```jsx
useEffect(() => {
  fetchStatus();
  const interval = setInterval(fetchStatus, 10000); // refresh every 10 seconds
  return () => clearInterval(interval); // cleanup when leaving the page
}, [doctorId]);
```

#### Why is this pattern so crucial?
1. **Immediate Execution**: Calling `fetchStatus()` immediately loads the queue data on component mount (no 10-second delay for the patient).
2. **Periodic Polling**: `setInterval(fetchStatus, 10000)` re-runs `fetchStatus` every 10,000 milliseconds (10 seconds), keeping the token number and queue count updated while the patient stays on the page.
3. **The Cleanup Function (`clearInterval`)**:
   - In React, returning a function from `useEffect` declares a **cleanup hook**.
   - If the patient navigates away to another page (e.g. back to Home or to their Appointments), the interval timer is cancelled.
   - **Without cleanup**, the browser would keep firing HTTP requests in the background forever, causing serious memory leaks and wasted server bandwidth!
4. **Dependency `[doctorId]`**: If the user switches between doctor profiles, the previous interval is cleared and a new one starts for the new doctor.

---

### 3. The `fetchStatus` Function

```jsx
async function fetchStatus() {
  try {
    const res = await api.get(`/tokens/doctor/${doctorId}`);
    setStatus(res.data);
  } catch (err) {
    // Quietly ignore - a failed poll shouldn't break the page
  }
}
```

- Calls `GET /api/tokens/doctor/:doctorId`.
- On success: `setStatus(res.data)` updates state, triggering a re-render with the latest queue numbers.
- On error: Silently caught in `catch`. If a temporary network hiccup happens during polling, we avoid displaying an alarming error banner; the UI simply keeps showing the last known valid state until the next 10-second cycle succeeds.

---

### 4. Rendering the Live Status

```jsx
if (!status) return null;

return (
  <Section icon="fa-ticket" title="Live Token" className={className}>
    <div className="flex flex-wrap items-center justify-between gap-2 rounded-field border border-line bg-soft p-3.5">
      <div className="flex items-center gap-[9px] text-[15px] font-bold">
        <span className="inline-block size-2 shrink-0 animate-live rounded-full bg-brand" aria-hidden="true"></span>
        Now serving <b className="font-serif text-brand">#{status.nowServing}</b>
      </div>
      <div className="text-[12.5px] text-mute">
        {status.queueLength} waiting · ~{status.avgWaitMinutes} min per patient
      </div>
    </div>
    {/* ... Get Token Button / My Token Display ... */}
  </Section>
);
```

- **Guard clause (`if (!status) return null;`)**: Renders nothing until the first status fetch resolves, preventing undefined property errors.
- **Live Indicator**: The `<span className="animate-live">` creates a pulsing green beacon to signal real-time queue activity.
- **Queue Info**: Displays `#{status.nowServing}` and `status.queueLength` waiting, along with estimated time per patient.

---

## Part 5 — The Full End-to-End Data Flow

```
1. DoctorProfile loads & renders <TokenDisplay doctorId="64f8..." />
        │
        ▼
2. TokenDisplay mounts
   - Initializes: status = null
   - useEffect runs:
     a) Immediately calls fetchStatus()
     b) Sets up 10-second setInterval(fetchStatus, 10000)
        │
        ▼
   ┌────────────────────────────────────────────────────────────┐
   │ BROWSER SENDS HTTP GET                                      │
   │ GET /api/tokens/doctor/64f8... HTTP/1.1                    │
   │ Host: localhost:5173 (proxied to localhost:5000)           │
   └────────────────────────────────────────────────────────────┘
        │
        ▼
3. Express matches Route:
   - app.use('/api/tokens', tokenRoutes)
   - router.get('/doctor/:doctorId', asyncHandler(getTokenStatus))
   - req.params.doctorId = "64f8..."
        │
        ▼
4. Controller executes:
   - Doctor.findById("64f8...") -> gets doctor.tokenSystem.nowServing
   - Token.countDocuments({
       doctor: "64f8...",
       date: "2026-10-20",
       status: "waiting"
     }) -> counts waiting tokens
        │
        ▼
   ┌────────────────────────────────────────────────────────────┐
   │ BACKEND SENDS HTTP RESPONSE                                 │
   │ Status: 200 OK                                              │
   │ Body: {                                                     │
   │   "nowServing": 4,                                          │
   │   "queueLength": 3,                                         │
   │   "avgWaitMinutes": 10                                      │
   │ }                                                           │
   └────────────────────────────────────────────────────────────┘
        │
        ▼
5. Frontend receives response:
   - setStatus(res.data)
   - TokenDisplay re-renders:
     "Now serving #4 | 3 waiting · ~10 min per patient"
        │
        ▼
6. Every 10 seconds:
   - setInterval repeats steps 2 to 5 automatically.
        │
        ▼
7. User leaves page (unmount):
   - React invokes cleanup callback: clearInterval(interval)
   - Timer stops. No wasted requests or memory leaks!
```

---

## Part 6 — Architectural Insights: Polling vs WebSockets

| Aspect | HTTP Polling (`setInterval`) | WebSockets / SSE |
|---|---|---|
| **Used In** | Apna Doctor CG (`TokenDisplay.jsx`) | Real-time chat apps / Trading platforms |
| **Complexity** | Extremely simple (standard REST endpoint + `setInterval`) | Requires dedicated WebSocket server, connection state handling, reconnect logic |
| **Server Load** | Periodic HTTP requests (every 10s per open tab) | Persistent open TCP connection |
| **When to Choose** | Perfect for non-critical, slow-changing queues (doctors call patients every 10–15 mins) | Required when sub-second real-time latency is critical |

---

## Part 7 — Quick Self-Test

1. **Why does `todayString()` slice the ISO date string to 10 characters (`YYYY-MM-DD`)?**  
   *To group tokens by day, so patient queues and token counts reset cleanly every morning without needing a manual database purge.*

2. **Why does the controller use `Token.countDocuments(...)` instead of `(await Token.find(...)).length`?**  
   *`countDocuments` lets the MongoDB database count matching indexes directly on the server without transferring all documents over the network to Node.js memory.*

3. **What happens if you omit `return () => clearInterval(interval);` in `TokenDisplay`'s `useEffect`?**  
   *The timer continues running in the browser even after the user leaves the page, leading to a memory leak and continuing to send useless requests every 10 seconds.*
