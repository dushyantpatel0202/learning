# Lesson 2: POST /api/doctors (Creating a Doctor Profile & Doctor Code Sequence)

In **Lesson 1**, we explored how `POST /api/auth/session` exchanges Firebase credentials for our application's JWT and assigns the user's role (`patient` or `doctor`).  
Now in **Lesson 2**, an authenticated user with role `doctor` visits their dashboard for the first time. They need to configure their medical practice: clinic name, specialization, consultation charges, geographic coordinates, and profile photo.

To establish this practice profile in the database, the client dispatches **`POST /api/doctors`**.

---

## Roadmap of POST Requests
1. ✅ **POST /api/auth/session** — Firebase token verification & JWT session creation (`AuthContext.jsx` & `authController.js`)
2. 🎯 **POST /api/doctors** — Creating a doctor profile & doctorCode generation (`DoctorDashboard.jsx` & `doctorController.js`)
3. ⏳ **POST /api/appointments** — Booking an appointment with consultation slot & fees (`BookingModal.jsx` & `appointmentController.js`)
4. ⏳ **POST /api/tokens/doctor/:doctorId** — Generating a live queue token for today (`TokenDisplay.jsx` & `tokenController.js`)
5. ⏳ **POST /api/uploads/signature** — Minting a secure Cloudinary upload signature (`PhotoUploader.jsx` & `uploadController.js`)

---

## Part 1 — Backend: Route Definition & Middleware Chain

File: [backend/routes/doctors.js](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/backend/routes/doctors.js)

```javascript
const express = require('express');
const asyncHandler = require('../middleware/asyncHandler');
const { protect, doctorOnly } = require('../middleware/auth');
const { createDoctor } = require('../controllers/doctorController');

const router = express.Router();

router.post('/', protect, doctorOnly, asyncHandler(createDoctor));

module.exports = router;
```

### Line-by-Line Breakdown:
- `router.post('/', ...)`: Handles `POST` requests to the root path of the doctors resource (`/api/doctors`).
- `protect`: The first gatekeeper middleware. It extracts the JWT from the `Authorization: Bearer <token>` header, verifies the secret signature, and attaches the user document to `req.user`. If the token is missing or expired, it halts the request with `401 Unauthorized`.
- `doctorOnly`: The second gatekeeper (Role-Based Access Control / RBAC). It checks `if (req.user.role !== 'doctor')` and rejects patients or guests with `403 Forbidden`.
- `asyncHandler(createDoctor)`: Only if both middleware checks succeed is `createDoctor` invoked.

---

## Part 2 — Backend: Controller Logic & Sequential ID Generation

File: [backend/controllers/doctorController.js](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/backend/controllers/doctorController.js)

```javascript
const Doctor = require('../models/Doctor');
const { isAllowedPhotoUrl } = require('../utils/photoUrl');

async function getNextDoctorCode() {
  const highest = await Doctor.findOne().sort({ doctorCode: -1 });
  return highest ? highest.doctorCode + 1 : 101;
}

// POST /api/doctors
async function createDoctor(req, res) {
  const existing = await Doctor.findOne({ user: req.user.id });
  if (existing) {
    return res.status(400).json({ message: 'You already have a doctor profile. Use PUT to edit it.' });
  }

  if (!isAllowedPhotoUrl(req.body.photoUrl)) {
    return res.status(400).json({ message: 'Please upload your photo using the Choose photo button.' });
  }

  const doctorCode = await getNextDoctorCode();
  const doctor = await Doctor.create({ ...req.body, user: req.user.id, doctorCode });
  res.status(201).json(doctor);
}
```

### Granular Line-by-Line Breakdown:
1. `const existing = await Doctor.findOne({ user: req.user.id });`: Enforces a **1-to-1 relationship** between a user account and their doctor listing. A doctor cannot create multiple profiles.
2. `if (existing) return res.status(400)...`: If a profile is already found, the server commands the client to use `PUT /api/doctors/me` instead of `POST`.
3. `isAllowedPhotoUrl(req.body.photoUrl)`: Validates that the uploaded photo URL originates strictly from our authorized Cloudinary bucket path and prevents arbitrary external URLs or script injections.
4. `getNextDoctorCode()`: Queries the single doctor with the highest numeric `doctorCode` (`sort({ doctorCode: -1 })`) and adds `1`. If no doctors exist yet, it seeds the first code at `101`.
5. `user: req.user.id`: **Crucial security guarantee**. The owner ID is injected directly from the verified session token (`req.user.id`), ignoring whatever the client might have passed in `req.body.user`.
6. `res.status(201).json(doctor)`: Returns HTTP status code **201 Created** alongside the newly created Mongoose document.

---

## Part 3 — Frontend: The API Call

File: [frontend/src/pages/DoctorDashboard.jsx](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/frontend/src/pages/DoctorDashboard.jsx)

```javascript
// Inside DoctorDashboard.jsx
async function handleSubmit(e) {
  e.preventDefault();
  setSaveMsg('');
  
  const payload = {
    ...form,
    experienceYears: Number(form.experienceYears),
    latitude: Number(form.latitude),
    longitude: Number(form.longitude),
    charges: form.charges.map((c) => ({ service: c.service, fee: Number(c.fee) })),
    contact: { ...form.contact, phones: form.contact.phones.filter((p) => p.trim() !== '') },
  };

  try {
    if (profile) {
      // Existing doctor uses PUT
      const res = await api.put('/doctors/me', payload);
      setProfile(res.data);
      setSaveMsg('Profile updated.');
    } else {
      // New doctor uses POST
      const res = await api.post('/doctors', payload);
      setProfile(res.data);
      setSaveMsg(`Profile created! Your doctor code is #${res.data.doctorCode} - patients can search this directly.`);
      loadAppointments();
      loadQueue(res.data._id);
    }
  } catch (err) {
    setSaveMsg(err.response?.data?.message || 'Could not save profile.');
  }
}
```

### Line-by-Line Breakdown:
- `payload sanitization`: Converts text field numbers (e.g. `experienceYears`, `latitude`, `longitude`, `fee`) into true JavaScript numbers (`Number(...)`), ensuring strict Mongoose schema compatibility.
- `const res = await api.post('/doctors', payload)`: Dispatches the HTTP POST request. Axios automatically serializes `payload` as JSON and includes the `Authorization` header.
- `setProfile(res.data)`: Storing `res.data` in state transitions the dashboard from the initial setup mode into the full clinical management cockpit (displaying queue controls and appointments).
- `loadQueue(res.data._id)`: Initializes the live queue polling for the new doctor profile ID.

---

## Part 4 — Frontend: Form State & Controlled Inputs

The `DoctorDashboard` component maintains form state with all necessary profile fields:

```javascript
const [form, setForm] = useState({
  name: '',
  specialization: 'General Physician',
  clinicName: '',
  address: '',
  city: '',
  district: 'Dehradun',
  charges: [{ service: 'Consultation', fee: 300 }],
  experienceYears: 5,
  latitude: 30.3165,
  longitude: 78.0322,
  photoUrl: '',
  contact: { phones: [''], email: '' },
});
```

When the user enters their information and clicks **"Create Profile"**, the form handler triggers the POST request and renders feedback immediately upon completion.

---

## Part 5 — The Full End-to-End Data Flow

```
[Doctor in DoctorDashboard.jsx]
       │
       ▼ 1. Fills clinic details, fee schedule, and clicks "Create Profile"
[Form Submit Handler]
       │
       ▼ 2. Transforms strings to numbers and constructs JSON payload
       │
       ▼ 3. api.post('/doctors', payload)
            Headers:
              Authorization: Bearer eyJhbGci...
              Content-Type: application/json
            Body:
              { name: "Dr. Aarti", specialization: "Dentist", fee: 500, ... }
       │
       ▼ 4. Express Server & Middlewares:
            - cors() & express.json()
            - protect: Verifies JWT, extracts req.user.id
            - doctorOnly: Verifies req.user.role === 'doctor'
       │
       ▼ 5. doctorController.createDoctor runs:
            - Checks Doctor.findOne({ user: req.user.id }) -> null (no existing profile)
            - Checks isAllowedPhotoUrl(req.body.photoUrl) -> true
            - getNextDoctorCode() -> scans DB, finds max 104 -> returns 105
            - Doctor.create({ ...req.body, user: req.user.id, doctorCode: 105 })
       │
       ▼ 6. MongoDB inserts document and returns full record
       │
       ▼ 7. Express Responds:
            HTTP 201 Created
            Body: { _id: "651a...", doctorCode: 105, name: "Dr. Aarti", ... }
       │
       ▼ 8. Frontend Axios receives 201 response:
            - setProfile(res.data) updates state
            - Screen switches from blank setup form to active practice dashboard
            - Displays message: "Profile created! Your doctor code is #105"
```

---

## Part 6 — Quick Self-Test

1. **Why does the controller call `Doctor.findOne({ user: req.user.id })` before creating a doctor?**  
   To enforce a strict one-to-one constraint: each user account can only own one doctor listing. If they already have one, they must update it via `PUT /api/doctors/me`.

2. **Why is `res.status(201)` returned instead of `res.status(200)`?**  
   In RESTful API conventions, `200 OK` indicates a successful general operation, whereas `201 Created` specifically signifies that a new persistent resource was successfully created on the server.

3. **Why is `doctorCode` generated on the server instead of being selected by the doctor?**  
   To prevent collisions, fraud, and race conditions. The backend guarantees that every doctor receives a unique, sequential numeric code (e.g., 101, 102, 103) that patients can reliably search from the homepage.
