# Lesson 2: PUT /api/appointments/:id/status (Updating Appointment Lifecycle Status)

In **Lesson 1**, we explored how doctors update their practice profiles via `PUT /api/doctors/me`.  
Now in **Lesson 2**, we explore how doctors manage their schedule in [DoctorDashboard.jsx](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/frontend/src/pages/DoctorDashboard.jsx).

When patients book appointments, they arrive with status `'pending'`. The doctor reviews their patient schedule and clicks **"Confirm"**, **"Complete"**, or **"Cancel"**. This fires **`PUT /api/appointments/:id/status`** to transition the appointment through its lifecycle.

---

## Roadmap of PUT Requests
1. ✅ **PUT /api/doctors/me** — Updating doctor profile details & photo (`DoctorDashboard.jsx` & `doctorController.js`)
2. 🎯 **PUT /api/appointments/:id/status** — Confirming, completing, or cancelling appointments (`DoctorDashboard.jsx` & `appointmentController.js`)
3. ⏳ **PUT /api/tokens/doctor/:doctorId/next** — Calling next patient in live queue (`DoctorDashboard.jsx` & `tokenController.js`)

---

## Part 1 — Backend: Route Definition & RBAC

File: [backend/routes/appointments.js](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/backend/routes/appointments.js)

```javascript
const express = require('express');
const asyncHandler = require('../middleware/asyncHandler');
const { protect, doctorOnly } = require('../middleware/auth');
const { updateAppointmentStatus } = require('../controllers/appointmentController');

const router = express.Router();

router.put('/:id/status', protect, doctorOnly, asyncHandler(updateAppointmentStatus));

module.exports = router;
```

### Line-by-Line Breakdown:
- `router.put('/:id/status', ...)`: Uses a targeted sub-resource route. It targets a specific appointment ID (`:id`) and explicitly limits the update scope to its `status`.
- `protect`: Verifies that the incoming request is authenticated with a valid JWT.
- `doctorOnly`: Ensures that only medical practitioners can alter appointment statuses (patients cannot confirm or mark their own appointments completed).

---

## Part 2 — Backend: Controller Logic

File: [backend/controllers/appointmentController.js](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/backend/controllers/appointmentController.js)

```javascript
// PUT /api/appointments/:id/status
async function updateAppointmentStatus(req, res) {
  const { status } = req.body;

  const appointment = await Appointment.findByIdAndUpdate(
    req.params.id,
    { $set: { status } },
    { new: true }
  );

  if (!appointment) {
    return res.status(404).json({ message: 'Appointment not found.' });
  }

  res.json(appointment);
}
```

### Granular Line-by-Line Breakdown:
1. `const { status } = req.body;`: Extracts the intended target status string (e.g. `'confirmed'`, `'completed'`, or `'cancelled'`).
2. `Appointment.findByIdAndUpdate(...)`:
   - `req.params.id`: Finds the target appointment by its MongoDB `_id` from the URL path.
   - `{ $set: { status } }`: Uses MongoDB's atomic `$set` operator to update only the `status` field, preserving the patient ID, doctor ID, date, time slot, and consultation fee untouched.
   - `{ new: true }`: Returns the freshly updated document.
3. `if (!appointment) return res.status(404)...`: Rejects invalid IDs with `404 Not Found`.
4. `res.json(appointment)`: Returns the updated document with status code `200 OK`.

---

## Part 3 — Frontend: Optimistic / Local Array Updates

File: [frontend/src/pages/DoctorDashboard.jsx](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/frontend/src/pages/DoctorDashboard.jsx)

```javascript
async function updateAppointmentStatus(id, status) {
  try {
    await api.put(`/appointments/${id}/status`, { status });
    // Update local state without needing to re-fetch the entire schedule!
    setAppointments((prev) =>
      prev.map((a) => (a._id === id ? { ...a, status } : a))
    );
  } catch (err) {
    // Graceful error handling
  }
}
```

### UI Action Buttons:
```javascript
<div className="flex gap-2">
  {apt.status === 'pending' && (
    <button
      onClick={() => updateAppointmentStatus(apt._id, 'confirmed')}
      className="text-xs px-2.5 py-1 bg-brand text-white rounded-md"
    >
      Confirm
    </button>
  )}
  {apt.status === 'confirmed' && (
    <button
      onClick={() => updateAppointmentStatus(apt._id, 'completed')}
      className="text-xs px-2.5 py-1 bg-emerald-600 text-white rounded-md"
    >
      Mark Done
    </button>
  )}
  {apt.status !== 'cancelled' && apt.status !== 'completed' && (
    <button
      onClick={() => updateAppointmentStatus(apt._id, 'cancelled')}
      className="text-xs px-2.5 py-1 bg-red-100 text-red-600 rounded-md"
    >
      Cancel
    </button>
  )}
</div>
```

### Key UI Principles:
- **Targeted Array Mapping**: Notice `setAppointments((prev) => prev.map(...))`. Rather than firing an expensive `GET /api/appointments/doctor` call to re-query the entire list over the network, React updates only the matching item in state array.
- **Conditional Action Buttons**: The UI buttons adapt dynamically based on the current appointment status (e.g. "Confirm" disappears once confirmed, replaced by "Mark Done").

---

## Part 4 — The Full End-to-End Data Flow

```
[Doctor in DoctorDashboard]
       │
       ▼ 1. Clicks "Confirm" next to an appointment for Aryan Sharma
       │
       ▼ 2. updateAppointmentStatus(id, "confirmed") executes
       │
       ▼ 3. HTTP PUT /api/appointments/652f10b.../status
            Headers:
              Authorization: Bearer <doctor_token>
              Content-Type: application/json
            Body:
              { "status": "confirmed" }
       │
       ▼ 4. Express Middlewares:
            - protect: Verifies doctor's JWT
            - doctorOnly: Verifies doctor role
       │
       ▼ 5. Controller executes Appointment.findByIdAndUpdate:
            - MongoDB updates appointment.status = "confirmed"
            - Returns updated appointment
       │
       ▼ 6. Response: HTTP 200 OK
            Body: { _id: "652f10b...", status: "confirmed", ... }
       │
       ▼ 7. Frontend receives 200:
            - React prev.map updates item status to "confirmed"
            - Badge turns green ("Confirmed")
            - Patient sees the updated status in their own dashboard
```

---

## Part 5 — Quick Self-Test

1. **Why is the endpoint named `PUT /api/appointments/:id/status` rather than `PUT /api/appointments/:id`?**  
   Targeted sub-endpoints make the API's contract explicit. By scoping the route to `/status`, it communicates that only the lifecycle status is being transitioned, preventing accidental modification of dates, fees, or patient links.

2. **How does the frontend avoid a flickering screen or full page reload when changing an appointment's status?**  
   By updating local React state via `setAppointments((prev) => prev.map(...))` upon receiving the successful HTTP 200 response, updating only the affected card in place.

3. **What happens to the patient's view when this PUT request succeeds?**  
   When the patient visits their dashboard (`GET /api/appointments/my`), the appointment displays with status `'confirmed'` instead of `'pending'`.
