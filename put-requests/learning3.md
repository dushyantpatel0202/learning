# Lesson 3: PUT /api/tokens/doctor/:doctorId/next (Advancing Live Queue & Calling Next Patient)

In **Lesson 2**, we learned how doctors manage appointments via `PUT /api/appointments/:id/status`.  
Now in **Lesson 3**, we study one of the most dynamic real-time operations in the application: **Advancing the Live Patient Queue**.

As patients wait in the clinic lobby, the doctor presses the **"Call Next Patient"** button inside [DoctorDashboard.jsx](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/frontend/src/pages/DoctorDashboard.jsx).  
This issues **`PUT /api/tokens/doctor/:doctorId/next`**, which executes a coordinated multi-step database transition: marking the current patient `'done'`, incrementing the doctor's `nowServing` counter, and transitioning the earliest waiting patient to `'serving'`.

---

## Roadmap of PUT Requests
1. ✅ **PUT /api/doctors/me** — Updating doctor profile details & photo (`DoctorDashboard.jsx` & `doctorController.js`)
2. ✅ **PUT /api/appointments/:id/status** — Confirming, completing, or cancelling appointments (`DoctorDashboard.jsx` & `appointmentController.js`)
3. 🎯 **PUT /api/tokens/doctor/:doctorId/next** — Calling next patient in live queue (`DoctorDashboard.jsx` & `tokenController.js`)

---

## Part 1 — Backend: Route Definition

File: [backend/routes/tokens.js](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/backend/routes/tokens.js)

```javascript
const express = require('express');
const asyncHandler = require('../middleware/asyncHandler');
const { protect, doctorOnly } = require('../middleware/auth');
const { callNextPatient } = require('../controllers/tokenController');

const router = express.Router();

router.put('/doctor/:doctorId/next', protect, doctorOnly, asyncHandler(callNextPatient));

module.exports = router;
```

### Line-by-Line Breakdown:
- `router.put('/doctor/:doctorId/next', ...)`: Listens for PUT requests instructing the server to advance the queue for `:doctorId`.
- `protect`: Verifies the doctor's authentication token.
- `doctorOnly`: Restricts queue management to doctors. Patients cannot call the next patient.

---

## Part 2 — Backend: Controller Logic & Atomic Queue State Transitions

File: [backend/controllers/tokenController.js](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/backend/controllers/tokenController.js)

```javascript
// PUT /api/tokens/doctor/:doctorId/next
async function callNextPatient(req, res) {
  // 1. Verify that this doctor account owns this profile
  const doctor = await Doctor.findOne({ _id: req.params.doctorId, user: req.user.id });
  if (!doctor) {
    return res.status(404).json({ message: 'Doctor profile not found for this account.' });
  }

  // 2. Mark any currently serving token as 'done'
  await Token.updateMany(
    { doctor: doctor._id, date: todayString(), status: 'serving' },
    { $set: { status: 'done' } }
  );

  // 3. Increment the doctor's public nowServing counter
  doctor.tokenSystem.nowServing += 1;
  await doctor.save();

  // 4. Find the earliest waiting token and mark it as 'serving'
  const nextToken = await Token.findOneAndUpdate(
    { doctor: doctor._id, date: todayString(), status: 'waiting' },
    { $set: { status: 'serving' } },
    { sort: { tokenNumber: 1 }, new: true }
  );

  res.json({
    nowServing: doctor.tokenSystem.nowServing,
    nowServingToken: nextToken
  });
}
```

### Granular Line-by-Line Breakdown:
1. `Doctor.findOne({ _id: req.params.doctorId, user: req.user.id })`: **Ownership Enforcement**. A doctor cannot call patients for another doctor's queue.
2. `Token.updateMany({ doctor: doctor._id, date: todayString(), status: 'serving' }, { $set: { status: 'done' } })`: Closes out previous consultations so no two patients are listed as simultaneously inside the consultation room.
3. `doctor.tokenSystem.nowServing += 1; await doctor.save();`: Updates the public live board number so all waiting patients polling the queue immediately see the counter advance (e.g. from `#3` to `#4`).
4. `Token.findOneAndUpdate(..., { sort: { tokenNumber: 1 }, new: true })`:
   - `{ status: 'waiting' }`: Selects from patients who are currently waiting outside.
   - `sort: { tokenNumber: 1 }`: **First-Come, First-Served (FIFO)** ordering. Picks the lowest token number waiting.
   - `{ $set: { status: 'serving' } }`: Promotes their token to the active consultation state.
5. `res.json(...)`: Returns the updated `nowServing` number and the full `nextToken` object.

---

## Part 3 — Frontend: Trigger & State Synchronization

File: [frontend/src/pages/DoctorDashboard.jsx](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/frontend/src/pages/DoctorDashboard.jsx)

```javascript
async function callNextPatient() {
  if (!profile) return;
  setQueueBusy(true); // Disable button to prevent double-clicks
  try {
    const res = await api.put(`/tokens/doctor/${profile._id}/next`);
    
    // 1. Immediately update nowServing on profile in state
    setProfile((prev) => ({
      ...prev,
      tokenSystem: { ...prev.tokenSystem, nowServing: res.data.nowServing },
    }));

    // 2. Re-fetch queue to update waiting vs serving status chips
    loadQueue(profile._id);
  } catch (err) {
    // Graceful error handling
  } finally {
    setQueueBusy(false);
  }
}
```

### Line-by-Line Breakdown:
- `setQueueBusy(true)`: Immediately disables the "Call Next Patient" button while the network request is in flight. This eliminates accidental double clicks that would skip patients.
- `setProfile((prev) => ...)`: Performs an immediate state update with `res.data.nowServing`, updating the prominent large number on the doctor's screen without waiting for other requests.
- `loadQueue(profile._id)`: Refreshes the queue list so the patient card visually transitions from the "Waiting" section to the active "Currently Consulting" card.

---

## Part 4 — The Full End-to-End Data Flow

```
[Doctor in DoctorDashboard]
       │
       ▼ 1. Clicks "Call Next Patient"
       │
       ▼ 2. setQueueBusy(true) disables button
       │
       ▼ 3. HTTP PUT /api/tokens/doctor/651a.../next
            Headers:
              Authorization: Bearer <doctor_token>
       │
       ▼ 4. Express Server & Middlewares:
            - protect: Verifies doctor's JWT
            - doctorOnly: Verifies role === 'doctor'
       │
       ▼ 5. callNextPatient controller:
            - Checks Doctor ownership: _id == req.params.doctorId && user == req.user.id
            - Token.updateMany(status: "serving" -> "done")
            - doctor.tokenSystem.nowServing (3 -> 4)
            - Token.findOneAndUpdate(status: "waiting", sort: { tokenNumber: 1 }) -> sets status: "serving"
       │
       ▼ 6. Response: HTTP 200 OK
            Body: { nowServing: 4, nowServingToken: { tokenNumber: 4, patient: { ... } } }
       │
       ▼ 7. Frontend State Update:
            - doctor.tokenSystem.nowServing is now 4
            - loadQueue() fetches updated list
            - Token #3 moves to Done
            - Token #4 highlights as Now Serving
       │
       ▼ 8. Waiting Patients on Homepage / TokenDisplay:
            - Polling interval triggers GET /api/tokens/doctor/:doctorId
            - Sees nowServing changed from #3 to #4
            - Patient holding token #4 gets alert: "It's your turn!"
```

---

## Part 5 — Quick Self-Test

1. **Why does the controller use `Token.updateMany(..., { status: 'done' })` before finding the next waiting token?**  
   To ensure data consistency. Before a new patient is called into the consultation room, any previous patient whose status was `'serving'` must be finalized as `'done'`.

2. **Why does `findOneAndUpdate` sort by `{ tokenNumber: 1 }`?**  
   To guarantee a strict First-In, First-Out (FIFO) queue order, ensuring the patient who arrived first and holds the lowest token number is called next.

3. **How does this PUT request communicate with patients waiting outside?**  
   It updates the doctor's `tokenSystem.nowServing` in MongoDB. Patient devices run polling intervals via `GET /api/tokens/doctor/:doctorId`. When the next poll executes, they receive the updated `nowServing` number and their UI updates automatically.
