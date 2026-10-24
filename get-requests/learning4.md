# Lesson 4: GET /api/appointments/my & GET /api/tokens/my (Patient's Appointments & Active Tokens)

In **Lesson 3**, we saw how `GET /api/tokens/doctor/:doctorId` allows any visitor to view a doctor's live queue.  
Now in **Lesson 4**, we turn to the patient's personal control center: **[PatientAppointments.jsx](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/frontend/src/pages/PatientAppointments.jsx)**.

Here, patients need to see all their booked consultations and queue tokens across all doctors in one unified dashboard. We will explore:
1. **Protected routes** requiring JWT authentication (`protect` middleware).
2. **Concurrent fetching** using `Promise.all` in React.
3. **Mongoose population** (`.populate()`) linking tokens/appointments back to doctor records.
4. **Backend-derived metrics** (calculating real-time estimated wait minutes).

---

## Roadmap of GET Requests
1. ✅ **GET /api/doctors** — Fetching doctors list with search and filters (`Home.jsx`)
2. ✅ **GET /api/doctors/:id** — Fetching a single doctor's complete profile (`DoctorProfile.jsx`)
3. ✅ **GET /api/tokens/doctor/:doctorId** — Live token and queue status for a doctor (`TokenDisplay.jsx`)
4. 🎯 **GET /api/appointments/my & GET /api/tokens/my** — Patient's appointments and tokens (`PatientAppointments.jsx`)
5. ⏳ **GET /api/doctors/me/profile** — Logged-in doctor's profile (`DoctorDashboard.jsx`)
6. ⏳ **GET /api/tokens/doctor/:doctorId/queue & GET /api/appointments/doctor** — Doctor's live patient queue & appointments list (`DoctorDashboard.jsx`)

---

## Part 1 — Backend: Route Definitions & Auth Middleware

### 1. Route Registrations

In **[backend/routes/tokens.js](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/backend/routes/tokens.js#L14)**:
```javascript
router.get('/my', protect, asyncHandler(getMyTokens));
```
*(Combined with app-level prefix `/api/tokens`, becomes `GET /api/tokens/my`)*

In **[backend/routes/appointments.js](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/backend/routes/appointments.js#L14)**:
```javascript
router.get('/my', protect, asyncHandler(getMyAppointments));
```
*(Combined with app-level prefix `/api/appointments`, becomes `GET /api/appointments/my`)*

---

### 2. The `protect` Authentication Middleware

**File:** [backend/middleware/auth.js](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/backend/middleware/auth.js#L8-L26)

```javascript
function protect(req, res, next) {
  const authHeader = req.headers.authorization;

  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    return res.status(401).json({ message: 'No token provided. Please log in.' });
  }

  const token = authHeader.split(' ')[1];

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded; // { id: '64f8...', role: 'patient' }
    next();
  } catch (err) {
    return res.status(401).json({ message: 'Invalid or expired token.' });
  }
}
```

### Line-by-Line Breakdown:
- **`req.headers.authorization`**: The browser's Axios client automatically attaches `Authorization: Bearer <token>` via Axios request interceptors whenever a patient logs in.
- **`jwt.verify(token, process.env.JWT_SECRET)`**: Verifies the signature of the token. If the token was forged, tampered with, or expired, an exception is thrown and Express halts the request with an HTTP **401 Unauthorized**.
- **`req.user = decoded`**: Attaches the decoded payload (containing user ID and role) to the `req` object. The downstream controller can now reliably know who is making the request via `req.user.id`.
- **`next()`**: Hands control to the controller function (`getMyTokens` or `getMyAppointments`).

---

## Part 2 — Backend: Controller Logic

### 1. Appointments Controller (`getMyAppointments`)

**File:** [backend/controllers/appointmentController.js](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/backend/controllers/appointmentController.js#L24-L30)

```javascript
// GET /api/appointments/my
async function getMyAppointments(req, res) {
  const appointments = await Appointment.find({ patient: req.user.id })
    .populate('doctor', 'name specialization clinicName city')
    .sort({ date: 1 });
  res.json(appointments);
}
```

#### Line-by-Line Breakdown:
- **`Appointment.find({ patient: req.user.id })`**: Queries the `appointments` collection filtering exclusively by the authenticated user's ID. Patients can never view other patients' appointments.
- **`.populate('doctor', 'name specialization clinicName city')`**:
  - The `Appointment` model stores only the `ObjectId` reference to the doctor (`doctor: { type: ObjectId, ref: 'Doctor' }`).
  - Mongoose joins the `doctors` collection and replaces the raw `ObjectId` with the doctor's document, selectively including only `name`, `specialization`, `clinicName`, and `city`.
- **`.sort({ date: 1 })`**: Chronologically sorts upcoming appointments so the earliest upcoming date appears first.

---

### 2. Tokens Controller (`getMyTokens`)

**File:** [backend/controllers/tokenController.js](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/backend/controllers/tokenController.js#L86-L100)

```javascript
// GET /api/tokens/my
async function getMyTokens(req, res) {
  const tokens = await Token.find({ patient: req.user.id })
    .populate('doctor', 'name specialization photoUrl tokenSystem doctorCode')
    .sort({ createdAt: -1 });

  const withEstimate = tokens.map((t) => {
    const nowServing = t.doctor?.tokenSystem?.nowServing ?? 0;
    const avgWait = t.doctor?.tokenSystem?.avgWaitMinutes ?? 10;
    const peopleAhead = Math.max(t.tokenNumber - nowServing, 0);
    return { ...t.toObject(), estimatedWaitMinutes: peopleAhead * avgWait };
  });

  res.json(withEstimate);
}
```

#### Line-by-Line Breakdown:
- **`Token.find({ patient: req.user.id })`**: Retrieves all tokens generated by this patient, sorted newest first (`{ createdAt: -1 }`).
- **`.populate('doctor', 'name specialization photoUrl tokenSystem doctorCode')`**: Loads the doctor's profile and importantly their live `tokenSystem` (`nowServing` and `avgWaitMinutes`).
- **Dynamic Estimation Math**:
  - `peopleAhead = Math.max(t.tokenNumber - nowServing, 0)`: If your token is #7 and the doctor is now serving #4, there are 3 patients ahead of you.
  - `estimatedWaitMinutes = peopleAhead * avgWait`: `3 patients * 10 mins = 30 mins wait`.
  - Attaches `estimatedWaitMinutes` to the response object before sending it to the client!

---

## Part 3 — Frontend: Routing & Page Component

### 1. Route Registration in `frontend/src/App.jsx`

```jsx
<Route path="/my-appointments" element={<PatientAppointments />} />
```

---

### 2. Concurrent Fetching in `PatientAppointments.jsx`

**File:** [frontend/src/pages/PatientAppointments.jsx](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/frontend/src/pages/PatientAppointments.jsx#L11-L18)

```jsx
export default function PatientAppointments() {
  const [tokens, setTokens] = useState([]);
  const [appointments, setAppointments] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    Promise.all([api.get('/tokens/my'), api.get('/appointments/my')])
      .then(([tokenRes, apptRes]) => {
        setTokens(tokenRes.data);
        setAppointments(apptRes.data);
      })
      .finally(() => setLoading(false));
  }, []);
```

#### Why use `Promise.all`?
- **Sequential (Slow)**:
  `await api.get('/tokens/my')` (takes 200ms) then `await api.get('/appointments/my')` (takes 200ms) = **400ms total wait**.
- **Parallel with `Promise.all` (Fast)**:
  Both requests are dispatched over the network simultaneously. The total wait time is only max(200ms, 200ms) = **200ms**!
- **`.finally(() => setLoading(false))`**: Guarantees that whether requests succeed or fail, the loading spinner stops.

---

### 3. Rendering Tokens and Appointments Side-by-Side

**File:** [frontend/src/pages/PatientAppointments.jsx](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/frontend/src/pages/PatientAppointments.jsx#L28-L78)

```jsx
{!loading && (
  <div className="grid items-start gap-8 md:grid-cols-2 lg:gap-12">
    {/* Column 1: Live Tokens */}
    <section>
      <h3><i className="fas fa-ticket text-brand"></i> My Tokens</h3>
      {tokens.length === 0 && <p>You haven't generated any tokens yet.</p>}
      {tokens.map((t) => (
        <InfoBox key={t._id}>
          <div className="flex justify-between">
            <span>Token #{t.tokenNumber} — {t.doctor?.name}</span>
            <StatusBadge status={t.status} />
          </div>
          <div>
            {t.doctor?.specialization} · Now serving #{t.doctor?.tokenSystem?.nowServing ?? 0} · ~{t.estimatedWaitMinutes} min estimated wait
          </div>
          <Link to={`/doctor/${t.doctor._id}`}>View doctor's profile</Link>
        </InfoBox>
      ))}
    </section>

    {/* Column 2: Scheduled Appointments */}
    <section>
      <h3><i className="fas fa-calendar-check text-brand"></i> My Appointments</h3>
      {appointments.length === 0 && <p>You haven't booked any appointments yet.</p>}
      {appointments.map((appt) => (
        <InfoBox key={appt._id}>
          <div className="flex justify-between">
            <span>{appt.doctor?.name} — {appt.doctor?.specialization}</span>
            <StatusBadge status={appt.status} />
          </div>
          <div>
            {appt.service} · {appt.date} at {appt.timeSlot} · ₹{appt.fee}
          </div>
        </InfoBox>
      ))}
    </section>
  </div>
)}
```

---

## Part 4 — The Full End-to-End Data Flow

```
1. Patient navigates to "/my-appointments"
        │
        ▼
2. <PatientAppointments /> mounts
   - State initialized: tokens = [], appointments = [], loading = true
   - useEffect initiates Promise.all()
        │
        ├───────────────────────────────────────────────────────┐
        ▼                                                       ▼
   HTTP GET /api/tokens/my                                 HTTP GET /api/appointments/my
   Headers: Authorization: Bearer <jwt>                    Headers: Authorization: Bearer <jwt>
        │                                                       │
        ▼                                                       ▼
3. Express verifies JWT in 'protect':                   3. Express verifies JWT in 'protect':
   - req.user = { id: "user123", role: "patient" }         - req.user = { id: "user123", role: "patient" }
        │                                                       │
        ▼                                                       ▼
4. tokenController.getMyTokens:                         4. appointmentController.getMyAppointments:
   - Token.find({ patient: "user123" })                    - Appointment.find({ patient: "user123" })
   - .populate('doctor')                                   - .populate('doctor')
   - Computes: estimatedWaitMinutes                        - .sort({ date: 1 })
        │                                                       │
        ▼                                                       ▼
   Response 200 OK: [ { tokenNumber: 5, ... } ]            Response 200 OK: [ { date: "2026-10-25", ... } ]
        │                                                       │
        └───────────────────────────┬───────────────────────────┘
                                    ▼
5. Promise.all resolves both responses simultaneously
   - setTokens(tokenRes.data)
   - setAppointments(apptRes.data)
   - setLoading(false)
        │
        ▼
6. Component re-renders:
   - Displays Token cards with real-time wait estimation and status badges.
   - Displays Appointment cards with booked slots, fees, and service names.
```

---

## Part 5 — Quick Self-Test

1. **Why does `protect` check `authHeader.startsWith('Bearer ')`?**  
   *Because HTTP standards stipulate passing OAuth/JWT bearer credentials in the `Authorization: Bearer <token>` format.*

2. **What does Mongoose's `.populate('doctor', 'name specialization')` do?**  
   *It acts as an automatic foreign-key lookup, replacing the stored doctor `ObjectId` with the actual doctor document, returning only the `name` and `specialization` fields.*

3. **Why is `Promise.all` superior to using two independent `await` statements in `useEffect`?**  
   *`Promise.all` fires both network requests concurrently, cutting the total loading time in half.*
