# Lesson 6: GET /api/tokens/doctor/:doctorId/queue & GET /api/appointments/doctor (Doctor's Live Queue & Patient Schedule)

In **Lesson 5**, we explored how `GET /api/doctors/me/profile` loads the doctor's practice profile in **[DoctorDashboard.jsx](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/frontend/src/pages/DoctorDashboard.jsx)**.  
Now in **Lesson 6**, we complete the GET request roadmap by examining the doctor's **daily clinical cockpit**:
1. **`GET /api/tokens/doctor/:doctorId/queue`** — The real-time queue of patients waiting outside the clinic door today.
2. **`GET /api/appointments/doctor`** — The schedule of booked appointments with full patient contact details.

Together, these two endpoints give the practitioner complete visibility over their clinic workflow.

---

## Roadmap of GET Requests
1. ✅ **GET /api/doctors** — Fetching doctors list with search and filters (`Home.jsx`)
2. ✅ **GET /api/doctors/:id** — Fetching a single doctor's complete profile (`DoctorProfile.jsx`)
3. ✅ **GET /api/tokens/doctor/:doctorId** — Live token and queue status for a doctor (`TokenDisplay.jsx`)
4. ✅ **GET /api/appointments/my & GET /api/tokens/my** — Patient's appointments and tokens (`PatientAppointments.jsx`)
5. ✅ **GET /api/doctors/me/profile** — Logged-in doctor's profile (`DoctorDashboard.jsx`)
6. 🎯 **GET /api/tokens/doctor/:doctorId/queue & GET /api/appointments/doctor** — Doctor's live patient queue & appointments list (`DoctorDashboard.jsx`)

---

## Part 1 — Backend: Route Definitions & Access Control

### 1. Route Registrations

In **[backend/routes/tokens.js](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/backend/routes/tokens.js#L15)**:
```javascript
router.get('/doctor/:doctorId/queue', protect, doctorOnly, asyncHandler(getDoctorQueue));
```
*(Full path: `GET /api/tokens/doctor/:doctorId/queue`)*

In **[backend/routes/appointments.js](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/backend/routes/appointments.js#L15)**:
```javascript
router.get('/doctor', protect, doctorOnly, asyncHandler(getDoctorAppointments));
```
*(Full path: `GET /api/appointments/doctor`)*

### Security Principles:
- **`protect`**: Ensures request contains a valid JWT in `Authorization: Bearer <token>`.
- **`doctorOnly`**: Rejects any patient or unauthorized user with `403 Forbidden`.
- **HIPAA / Privacy Concern**: Why can't patients access these endpoints?
  Because these endpoints return sensitive patient information (patient names, phone numbers, appointment medical notes, and contact emails). Only verified doctors can view their own patient rosters.

---

## Part 2 — Backend: Controller Logic

### 1. Live Queue Controller (`getDoctorQueue`)

**File:** [backend/controllers/tokenController.js](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/backend/controllers/tokenController.js#L74-L84)

```javascript
// GET /api/tokens/doctor/:doctorId/queue
async function getDoctorQueue(req, res) {
  const doctor = await Doctor.findOne({ _id: req.params.doctorId, user: req.user.id });
  if (!doctor) return res.status(404).json({ message: 'Doctor profile not found for this account.' });

  const tokens = await Token.find({ doctor: doctor._id, date: todayString() })
    .populate('patient', 'name phone')
    .sort({ tokenNumber: 1 });

  res.json(tokens);
}
```

#### Line-by-Line Breakdown:
- **`Doctor.findOne({ _id: req.params.doctorId, user: req.user.id })`**:
  - **Crucial Multi-Tenant Ownership Check**: Notice that the query matches **both** `_id` AND `user: req.user.id`.
  - Even if Doctor X is authenticated, if Doctor X attempts to pass Doctor Y's ID in `:doctorId`, `findOne` will return `null`. This prevents doctors from snooping on rival clinics' queues!
- **`date: todayString()`**:
  - Only fetches tokens created for today. Historical tokens from yesterday are filtered out.
- **`.populate('patient', 'name phone')`**:
  - Replaces the foreign key `patient` with the patient's actual `name` and `phone` so the receptionist or doctor knows who is entering next.
- **`.sort({ tokenNumber: 1 })`**:
  - Orders tokens numerically (1, 2, 3...) so the queue reflects the exact arrival sequence.

---

### 2. Appointments Controller (`getDoctorAppointments`)

**File:** [backend/controllers/appointmentController.js](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/backend/controllers/appointmentController.js#L32-L41)

```javascript
// GET /api/appointments/doctor
async function getDoctorAppointments(req, res) {
  const doctor = await Doctor.findOne({ user: req.user.id });
  if (!doctor) return res.status(404).json({ message: 'No doctor profile found for this account.' });

  const appointments = await Appointment.find({ doctor: doctor._id })
    .populate('patient', 'name phone email')
    .sort({ date: 1 });
  res.json(appointments);
}
```

#### Line-by-Line Breakdown:
- **`Doctor.findOne({ user: req.user.id })`**:
  - Finds the doctor's record from the authenticated token without even needing a URL parameter.
- **`Appointment.find({ doctor: doctor._id })`**:
  - Retrieves all appointments scheduled for this doctor.
- **`.populate('patient', 'name phone email')`**:
  - Attaches the patient's contact information so the clinic staff can confirm or notify the patient.
- **`.sort({ date: 1 })`**:
  - Ascending date order puts upcoming appointments at the top.

---

## Part 3 — Frontend: `DoctorDashboard.jsx`

**File:** [frontend/src/pages/DoctorDashboard.jsx](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/frontend/src/pages/DoctorDashboard.jsx#L80-L96)

### 1. The Fetch Functions

```jsx
async function loadAppointments() {
  try {
    const res = await api.get('/appointments/doctor');
    setAppointments(res.data);
  } catch (err) {
    // ignore - no profile yet
  }
}

async function loadQueue(doctorId) {
  try {
    const res = await api.get(`/tokens/doctor/${doctorId}/queue`);
    setQueue(res.data);
  } catch (err) {
    // ignore
  }
}
```

- Triggered right after `loadProfile()` succeeds.
- Populates the `appointments` and `queue` arrays in component state.

---

### 2. The Queue UI & "Call Next Patient" Integration

In the dashboard, the live queue displays:
- The currently serving token number.
- The list of waiting patients:

```jsx
{queue.map((t) => (
  <div key={t._id} className="flex items-center justify-between py-2 border-b">
    <div>
      <span className="font-bold">Token #{t.tokenNumber}</span> — {t.patient?.name || 'Walk-in'}
      <span className="text-xs text-mute ml-2">({t.patient?.phone})</span>
    </div>
    <StatusBadge status={t.status} />
  </div>
))}
```

When the doctor clicks **"Call Next Patient"**:
1. Sends `PUT /api/tokens/doctor/:doctorId/next`.
2. Marks the previous patient as `done` and advances `nowServing += 1`.
3. Calls `loadQueue(doctorId)` immediately to refresh the list!

---

### 3. The Appointments List & Status Management

```jsx
{appointments.map((appt) => (
  <InfoBox key={appt._id}>
    <div className="flex justify-between">
      <span>{appt.patient?.name} ({appt.patient?.phone})</span>
      <StatusBadge status={appt.status} />
    </div>
    <div>{appt.service} · {appt.date} at {appt.timeSlot}</div>
  </InfoBox>
))}
```

Doctors can review incoming bookings, inspect notes, and update statuses (e.g. `confirmed`, `completed`, `cancelled`).

---

## Part 4 — The Full End-to-End Data Flow

```
1. Doctor opens /doctor-dashboard
        │
        ▼
2. loadProfile() resolves -> retrieves doctor ID "doc_abc_123"
        │
        ├───────────────────────────────────────────────────────┐
        ▼                                                       ▼
   HTTP GET /api/tokens/doctor/doc_abc_123/queue           HTTP GET /api/appointments/doctor
   Headers: Bearer <jwt>                                   Headers: Bearer <jwt>
        │                                                       │
        ▼                                                       ▼
3. Express pipeline:                                    3. Express pipeline:
   - protect (verifies token)                              - protect (verifies token)
   - doctorOnly (verifies role === 'doctor')               - doctorOnly (verifies role === 'doctor')
        │                                                       │
        ▼                                                       ▼
4. tokenController.getDoctorQueue:                      4. appointmentController.getDoctorAppointments:
   - Doctor.findOne({                                      - Doctor.findOne({ user: req.user.id })
       _id: "doc_abc_123",                                 - Appointment.find({ doctor: doctor._id })
       user: req.user.id                                   - .populate('patient', 'name phone email')
     }) -> Ownership Verified!                             - .sort({ date: 1 })
   - Token.find({ doctor, date: todayString() })                │
   - .populate('patient', 'name phone')                         │
   - .sort({ tokenNumber: 1 })                                  │
        │                                                       │
        ▼                                                       ▼
   Response 200 OK:                                        Response 200 OK:
   [                                                       [
     { tokenNumber: 1, status: "serving",                    { patient: { name: "Ravi", phone: "987..." },
       patient: { name: "Aman", phone: "982..." } },           service: "Checkup", date: "2026-10-21" },
     { tokenNumber: 2, status: "waiting",                  ]
       patient: { name: "Pooja", phone: "912..." } }
   ]
        │                                                       │
        └───────────────────────────┬───────────────────────────┘
                                    ▼
5. React sets queue and appointments state:
   - Doctor sees live waiting room list with patient names and contact numbers.
   - Doctor sees complete upcoming consultation schedule.
```

---

## Part 5 — Quick Self-Test

1. **Why does `getDoctorQueue` query `Doctor.findOne({ _id: req.params.doctorId, user: req.user.id })` with both fields?**  
   *To guarantee ownership. This ensures that an authenticated doctor cannot view another doctor's private patient queue by simply modifying the `doctorId` in the URL.*

2. **Why do we populate `'name phone'` for the doctor's queue, but not in the public `GET /api/tokens/doctor/:doctorId` endpoint?**  
   *Privacy and HIPAA standards. The public endpoint only exposes anonymous numbers (`nowServing`, `queueLength`). Only the treating doctor has permission to see patient identities.*

3. **What keeps today's queue from showing tokens from yesterday?**  
   *The filter `{ date: todayString() }`, which restricts the MongoDB query strictly to tokens matching today's ISO date string (`YYYY-MM-DD`).*

my questions 

1. what is the difference between the tokens created for today and yesterday in tokens collection? 
2. there is any button for que on doctor dashboard for get request or it is automatically called when doctor log in? 
3. same for apointments? and should i add button for appointments get request? 
4.  for line 66 to 75 we can see user id and doctor id it prevent snooping it is ok but we know specific mongodb id so why we use user id and doctor id in this case?
5. we know that doctor id is specific mongodb id so why we use user id and doctor id in this case and it prevent snooping is just an excuse to use two id here??
6. i want to know on specific mongodb id we of doctor stores patient data who have token for consoltation or it only hold some id of patient
7. for appointment why we are not using two id like tokens  and what we are not using todaystring() in appointment? 
8. 

## Answers

1. A token document has a `date` field stored as a `YYYY-MM-DD` string. Tokens created on different days are separate MongoDB documents with different dates (and their own token numbers/statuses). When a patient requests a token, the app checks for an existing token for that same doctor, patient, and date. The doctor's queue endpoint filters by today's date, so yesterday's tokens remain stored but are not included in today's queue.

2. The queue GET request runs automatically when the dashboard loads: after the doctor profile is fetched, the frontend calls `loadQueue(profile._id)`. There is no separate button for fetching the queue. The "Call next patient" button is a different action: it sends a PUT request and then fetches the queue again. The queue is not continuously refreshed or polled, so it can become stale until one of those fetches happens.

3. Appointments are also fetched automatically after the doctor profile loads, through `loadAppointments()`, which calls `GET /api/appointments/doctor`. There is no separate fetch button. A button is not necessary for initial loading; it could be added as a manual refresh control if doctors need to reload new bookings without revisiting the dashboard. The current code does not automatically refresh the appointment list after another patient books.

4. The two fields have different jobs. `req.params.doctorId` identifies the doctor profile requested in the URL, while `req.user.id` comes from the authenticated doctor's verified login token and identifies the account that owns the profile. Matching both in `Doctor.findOne({ _id: ..., user: ... })` verifies that the requested profile belongs to the logged-in account. A MongoDB ID is an identifier, not proof that the requester is authorized to access that record.

5. It is not just an excuse to use two IDs: the query checks both resource identity and ownership. If the query checked only `_id`, knowing or guessing another doctor's ID could be enough to request that profile's private queue. The user ID alone can identify the current account's profile, but it would not verify that the doctor ID supplied in the URL is that same profile. For an endpoint without a doctor ID in the URL, such as `GET /api/appointments/doctor`, looking up the profile using the authenticated user is enough to find that account's own doctor profile.

6. A token document stores references (`ObjectId`s) to the doctor and patient; it does not copy the patient's name or phone into the token document. The queue controller uses `.populate('patient', 'name phone')` to look up those selected fields from the referenced User document when returning the response. Appointment documents likewise store doctor and patient references, while appointment-specific fields such as service, date, time slot, fee, and notes are stored on the appointment itself. The appointments endpoint populates selected patient fields for its response.

7. The appointments endpoint does not take a doctor ID from the URL. Instead, it finds the Doctor document belonging to the authenticated user (`Doctor.findOne({ user: req.user.id })`) and uses that profile's `_id` to find appointments. That is how it restricts results to the logged-in doctor's own appointments. It does not filter by `todayString()` because appointments are scheduled for particular dates and the dashboard currently loads the doctor's full appointment list, including past and future appointments. A separate "upcoming appointments" view could add a date filter, but it would be a different behavior.

8. No question was included for this number.
