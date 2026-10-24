# Lesson 1: PUT /api/doctors/me (Updating Doctor Profile & Photo)

Having mastered **GET** (data retrieval) and **POST** (resource creation), we now advance to **PUT** requests.  
In RESTful architecture, the **PUT** HTTP method represents **updating or replacing** an existing resource.

Our first PUT request is **`PUT /api/doctors/me`**, which allows a logged-in physician to modify their medical practice details — consultation fees, clinic address, phone numbers, or profile picture — safely and securely in [DoctorDashboard.jsx](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/frontend/src/pages/DoctorDashboard.jsx).

---

## Roadmap of PUT Requests
1. 🎯 **PUT /api/doctors/me** — Updating doctor profile details & photo (`DoctorDashboard.jsx` & `doctorController.js`)
2. ⏳ **PUT /api/appointments/:id/status** — Confirming, completing, or cancelling appointments (`DoctorDashboard.jsx` & `appointmentController.js`)
3. ⏳ **PUT /api/tokens/doctor/:doctorId/next** — Calling next patient in live queue (`DoctorDashboard.jsx` & `tokenController.js`)

---

## Part 1 — Backend: Route Definition

File: [backend/routes/doctors.js](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/backend/routes/doctors.js)

```javascript
const express = require('express');
const asyncHandler = require('../middleware/asyncHandler');
const { protect, doctorOnly } = require('../middleware/auth');
const { updateMyDoctor } = require('../controllers/doctorController');

const router = express.Router();

router.put('/me', protect, doctorOnly, asyncHandler(updateMyDoctor));

module.exports = router;
```

### Line-by-Line Breakdown:
- `router.put('/me', ...)`: Registers an HTTP route for `PUT` operations at `/api/doctors/me`.
- `protect`: Verifies the doctor's JWT session from `Authorization: Bearer <token>`.
- `doctorOnly`: Rejects requests from accounts with role `'patient'` with `403 Forbidden`.
- Notice the route path `/me`. Instead of accepting an arbitrary ID parameter like `/doctors/:id`, `/me` ensures the authenticated user can **only ever update their own profile**.

---

## Part 2 — Backend: Controller Logic & Mass Assignment Protection

File: [backend/controllers/doctorController.js](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/backend/controllers/doctorController.js)

```javascript
// PUT /api/doctors/me
async function updateMyDoctor(req, res) {
  // 1. Destructure to strip sensitive fields that must never be updated
  const { doctorCode, user, ...safeUpdates } = req.body;

  // 2. Validate photo URL if an update to photoUrl was provided
  if ('photoUrl' in safeUpdates) {
    const current = await Doctor.findOne({ user: req.user.id });
    if (!isAllowedPhotoUrl(safeUpdates.photoUrl, current?.photoUrl)) {
      return res.status(400).json({ message: 'Please upload your photo using the Choose photo button.' });
    }
  }

  // 3. Atomically find and update the document
  const doctor = await Doctor.findOneAndUpdate(
    { user: req.user.id },
    { $set: safeUpdates },
    { new: true, runValidators: true }
  );

  if (!doctor) {
    return res.status(404).json({ message: 'No doctor profile found for this account yet.' });
  }

  res.json(doctor);
}
```

### Granular Line-by-Line Breakdown:
1. `const { doctorCode, user, ...safeUpdates } = req.body;`: **Mass-Assignment Defense (Over-Posting)**. Even if an attacker attempts to modify `doctorCode: 999` or change the document owner `user: "attacker_id"`, the destructuring syntax strips them away into discarded variables, leaving only allowed practice attributes in `safeUpdates`.
2. `if ('photoUrl' in safeUpdates)`: Validates image domains only if the user modified the photo URL.
3. `Doctor.findOneAndUpdate({ user: req.user.id }, { $set: safeUpdates }, { new: true, runValidators: true })`:
   - `{ user: req.user.id }`: Scopes the query strictly to the logged-in doctor.
   - `{ $set: safeUpdates }`: MongoDB operator that modifies only the fields present in `safeUpdates` without overwriting untouched properties.
   - `{ new: true }`: Returns the **updated** document instead of the original pre-update document.
   - `{ runValidators: true }`: Forces Mongoose schema validators (e.g. min/max constraints, regex checks) to run on update operations (by default, Mongoose only validates on `.save()`).
4. `res.json(doctor)`: Responds with status `200 OK` and the updated doctor document.

---

## Part 3 — Frontend: Immediate Photo Updates & Form Submissions

File: [frontend/src/pages/DoctorDashboard.jsx](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/frontend/src/pages/DoctorDashboard.jsx)

In `DoctorDashboard.jsx`, the PUT request is dispatched in two distinct operational contexts:

### 1. Instant Photo Save upon Cloudinary Upload:
```javascript
async function handlePhotoUploaded(url) {
  updateField('photoUrl', url);
  if (profile) {
    // Existing doctor: save photo immediately so an uploaded photo is never lost!
    try {
      const res = await api.put('/doctors/me', { photoUrl: url });
      setProfile(res.data);
      setSaveMsg('Photo updated.');
    } catch (err) {
      setSaveMsg(err.response?.data?.message || 'Photo saving failed.');
    }
  }
}
```

### 2. Full Profile Details Submission:
```javascript
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
      const res = await api.put('/doctors/me', payload);
      setProfile(res.data);
      setSaveMsg('Profile updated.');
    }
  } catch (err) {
    setSaveMsg(err.response?.data?.message || 'Could not save profile.');
  }
}
```

### Line-by-Line Breakdown:
- **Instant Photo Sync**: As soon as Cloudinary returns a secure URL, the app issues a minimal PUT `{ photoUrl: url }`. This prevents user frustration where a doctor uploads a photo but loses it because they navigated away without pressing "Save Changes".
- **State Hydration**: When the backend responds with the updated document, `setProfile(res.data)` synchronizes React state, immediately updating the avatar, header title, and badges throughout the view.

---

## Part 4 — The Full End-to-End Data Flow

```
[Doctor in DoctorDashboard]
       │
       ▼ 1. Modifies fee from ₹400 to ₹500 and clicks "Save Changes"
       │
       ▼ 2. handleSubmit constructs payload object
       │
       ▼ 3. HTTP PUT /api/doctors/me
            Headers:
              Authorization: Bearer <doctor_token>
              Content-Type: application/json
            Body:
              {
                "charges": [{ "service": "Consultation", "fee": 500 }],
                "clinicName": "City Care Clinic",
                ...
              }
       │
       ▼ 4. Express Middlewares:
            - protect: Verifies token signature, extracts req.user.id
            - doctorOnly: Verifies req.user.role === 'doctor'
       │
       ▼ 5. updateMyDoctor controller:
            - Strips forbidden fields (doctorCode, user)
            - Executes Doctor.findOneAndUpdate({ user: req.user.id }, { $set: safeUpdates }, { new: true })
            - MongoDB updates document and returns fresh snapshot
       │
       ▼ 6. Response: HTTP 200 OK
            Body: { _id: "651a...", charges: [{ "service": "Consultation", "fee": 500 }], ... }
       │
       ▼ 7. Frontend receives updated doctor:
            - setProfile(res.data) updates state
            - Displays success indicator: "Profile updated."
```

---

## Part 5 — Quick Self-Test

1. **Why does the endpoint use `/api/doctors/me` instead of `/api/doctors/:doctorId`?**  
   Using `/me` derives the target doctor's identity strictly from the verified JWT token (`req.user.id`). This eliminates the risk of authorization bypass where a doctor might pass another doctor's ID in the URL to alter their listing.

2. **What does `{ new: true }` in `findOneAndUpdate` do?**  
   By default, Mongoose returns the document as it existed *before* the update was applied. Passing `{ new: true }` ensures Mongoose returns the updated document containing the new values, which the frontend can then display immediately.

3. **How does the backend prevent a doctor from tampering with their assigned `doctorCode`?**  
   Via destructuring: `const { doctorCode, user, ...safeUpdates } = req.body;`. Even if the client sends `{ doctorCode: 1 }` in the JSON body, it is discarded and excluded from the MongoDB `$set` update.
