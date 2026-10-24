# Lesson 5: GET /api/doctors/me/profile (Logged-in Doctor's Profile)

In **Lesson 4**, we explored the patient dashboard (`GET /api/appointments/my` & `GET /api/tokens/my`).  
Now in **Lesson 5**, we shift our perspective to the practitioner in **[DoctorDashboard.jsx](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/frontend/src/pages/DoctorDashboard.jsx)**.

When a physician logs in, the application needs to:
1. Verify their identity and enforce **Role-Based Access Control (RBAC)**.
2. Fetch their complete practice configuration (fees, clinic hours, phone numbers, geo-coordinates, verified status) via **`GET /api/doctors/me/profile`**.
3. Hydrate complex multi-field form state so the doctor can update their public listing anytime.

---

## Roadmap of GET Requests
1. ✅ **GET /api/doctors** — Fetching doctors list with search and filters (`Home.jsx`)
2. ✅ **GET /api/doctors/:id** — Fetching a single doctor's complete profile (`DoctorProfile.jsx`)
3. ✅ **GET /api/tokens/doctor/:doctorId** — Live token and queue status for a doctor (`TokenDisplay.jsx`)
4. ✅ **GET /api/appointments/my & GET /api/tokens/my** — Patient's appointments and tokens (`PatientAppointments.jsx`)
5. 🎯 **GET /api/doctors/me/profile** — Logged-in doctor's profile (`DoctorDashboard.jsx`)
6. ⏳ **GET /api/tokens/doctor/:doctorId/queue & GET /api/appointments/doctor** — Doctor's live patient queue & appointments list (`DoctorDashboard.jsx`)

---

## Part 1 — Backend: Route Definition & Two-Tier Middleware

**File:** [backend/routes/doctors.js](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/backend/routes/doctors.js#L23-L25)

```javascript
router.get('/', asyncHandler(getDoctors));
router.get('/me/profile', protect, doctorOnly, asyncHandler(getMyDoctorProfile));
router.get('/:id', asyncHandler(getDoctorById));
```

### Line-by-Line Breakdown:

- **`router.get('/me/profile', protect, doctorOnly, ...)`**:
  - Express supports chaining multiple middleware functions in series before reaching the controller.
  - The request must successfully pass **both** `protect` and `doctorOnly` before `getMyDoctorProfile` is invoked.
- **Route Order Matters**:
  - Notice `/me/profile` is declared **before** `/:id`.
  - If `/:id` were declared first, Express would capture the string `"me"` as `req.params.id` and invoke `getDoctorById` instead!
- **Difference between HTTP 401 and HTTP 403**:
  - **401 Unauthorized (`protect`)**: "We don't know who you are. Missing or invalid token."
  - **403 Forbidden (`doctorOnly`)**: "We know who you are, but you are a patient, not a doctor. You do not have permission to access this resource."

---

## Part 2 — Backend: The `doctorOnly` RBAC Middleware

**File:** [backend/middleware/auth.js](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/backend/middleware/auth.js#L28-L34)

```javascript
// Extra check for routes only doctors should use
function doctorOnly(req, res, next) {
  if (req.user.role !== 'doctor') {
    return res.status(403).json({ message: 'This action is for doctors only.' });
  }
  next();
}
```

- When the user registers or logs in, their JWT payload contains their role: `{ id: '...', role: 'doctor' }`.
- `protect` decodes the token and sets `req.user`.
- `doctorOnly` inspects `req.user.role`. If a patient tries to make a request to `/api/doctors/me/profile`, they are stopped immediately with HTTP 403.

---

## Part 3 — Backend: Controller Logic

**File:** [backend/controllers/doctorController.js](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/backend/controllers/doctorController.js#L76-L81)

```javascript
// GET /api/doctors/me/profile
async function getMyDoctorProfile(req, res) {
  const doctor = await Doctor.findOne({ user: req.user.id });
  if (!doctor) return res.status(404).json({ message: 'No doctor profile found for this account yet.' });
  res.json(doctor);
}
```

### Line-by-Line Breakdown:
- **`Doctor.findOne({ user: req.user.id })`**:
  - The `Doctor` schema stores a reference to the login account: `user: { type: ObjectId, ref: 'User' }`.
  - Instead of looking up by `_id`, this query looks up by the foreign key `user`.
  - This ensures a doctor can only ever access their own profile!
- **`if (!doctor) return res.status(404)...`**:
  - When a doctor registers for the first time, their `User` account exists, but they haven't filled out their clinic details yet.
  - Returning `404` informs the frontend that no profile exists yet, signaling the UI to display the empty creation form rather than an error banner.
- **`res.json(doctor)`**:
  - Sends the full doctor document containing clinic address, timings, charges, phone numbers, and coordinates.

---

## Part 4 — Frontend: `DoctorDashboard.jsx`

**File:** [frontend/src/pages/DoctorDashboard.jsx](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/frontend/src/pages/DoctorDashboard.jsx#L46-L78)

### 1. Client-Side Authentication Guard

```jsx
useEffect(() => {
  if (!user) {
    navigate('/login');
    return;
  }
  if (user.role !== 'doctor') {
    navigate('/');
    return;
  }
  loadProfile();
}, [user]);
```
- If a guest accesses `/doctor-dashboard`, they are redirected to `/login`.
- If a logged-in patient accesses it, they are redirected to `/` (home).
- Only verified doctor accounts trigger `loadProfile()`.

---

### 2. Loading Profile & Hydrating Form State

```jsx
async function loadProfile() {
  try {
    const res = await api.get('/doctors/me/profile');
    setProfile(res.data);
    setForm({
      ...res.data,
      latitude: res.data.latitude ?? '',
      longitude: res.data.longitude ?? '',
      contact: {
        ...res.data.contact,
        phones: res.data.contact?.phones?.length ? res.data.contact.phones : [''],
      },
    });
    loadAppointments();
    loadQueue(res.data._id);
  } catch (err) {
    setProfile(null); // New doctor: will display blank form
  } finally {
    setLoading(false);
  }
}
```

#### What happens here:
1. `api.get('/doctors/me/profile')` retrieves the doctor's document.
2. **Form Hydration**: The response object is unpacked into React's `form` state (`setForm(...)`), populating:
   - Text fields: `name`, `specialization`, `qualifications`, `clinicName`, `address`, `about`.
   - Nested timings: `timings.days`, `timings.hours`.
   - Charge tiers: `charges: [{ service: '...', fee: 300 }]`.
   - Multi-phone inputs: `contact.phones`.
3. **Chaining Next Steps**: Once `res.data._id` is retrieved, it kicks off:
   - `loadAppointments()` (`GET /api/appointments/doctor`)
   - `loadQueue(res.data._id)` (`GET /api/tokens/doctor/:doctorId/queue`)  
   *(These are the subjects of Lesson 6!)*

---

## Part 5 — The Full End-to-End Data Flow

```
1. Doctor logs into the application and opens "/doctor-dashboard"
        │
        ▼
2. <DoctorDashboard /> mounts:
   - Checks user.role === 'doctor'
   - Calls loadProfile()
        │
        ▼
   ┌────────────────────────────────────────────────────────┐
   │ BROWSER SENDS HTTP GET                                  │
   │ GET /api/doctors/me/profile HTTP/1.1                    │
   │ Headers: Authorization: Bearer <jwt_doctor_token>      │
   └────────────────────────────────────────────────────────┘
        │
        ▼
3. Express Middleware Pipeline:
   - Step A: protect verifies JWT -> req.user = { id: "doc_user_99", role: "doctor" }
   - Step B: doctorOnly checks req.user.role === 'doctor' -> Passes!
        │
        ▼
4. Controller executes:
   - Doctor.findOne({ user: "doc_user_99" })
   - MongoDB finds doctor record with doctorCode, clinicName, timings, etc.
        │
        ▼
   ┌────────────────────────────────────────────────────────┐
   │ BACKEND SENDS HTTP RESPONSE                             │
   │ Status: 200 OK                                          │
   │ Body: { _id: "doc123", name: "Dr. A. Verma", ... }      │
   └────────────────────────────────────────────────────────┘
        │
        ▼
5. Frontend sets profile and hydrates form state:
   - All profile form inputs pre-populate with existing settings.
   - Status indicators (Verified, Doctor Code #104) render.
   - Triggers loadAppointments() and loadQueue("doc123").
```

---

## Part 6 — Quick Self-Test

1. **Why does Express need `doctorOnly` in addition to `protect`?**  
   *`protect` only verifies that the user is logged in. Without `doctorOnly`, any authenticated patient could attempt to query or edit doctor profile routes.*

2. **Why does the controller query `Doctor.findOne({ user: req.user.id })` rather than `Doctor.findById(...)`?**  
   *Because `req.user.id` is the ID of the `User` account from the JWT. The `Doctor` collection references this through the `user` field.*

3. **Why is it essential to declare `router.get('/me/profile', ...)` before `router.get('/:id', ...)` in `doctors.js`?**  
   *Because Express matches routes in sequential declaration order; otherwise `:id` would capture `'me'` as a dynamic parameter.*
