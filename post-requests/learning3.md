# Lesson 3: POST /api/appointments (Booking a Medical Appointment)

In **Lesson 2**, we covered how doctors register their practices via `POST /api/doctors`.  
Now in **Lesson 3**, we focus on the patient's perspective. When a patient navigates to a doctor's profile ([DoctorProfile.jsx](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/frontend/src/pages/DoctorProfile.jsx)) and clicks **"Book Appointment"**, a controlled modal dialog ([BookingModal.jsx](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/frontend/src/components/BookingModal.jsx)) appears.

Submitting this modal triggers **`POST /api/appointments`**, which links the patient and the physician into a confirmed or pending booking in MongoDB.

---

## Roadmap of POST Requests
1. ✅ **POST /api/auth/session** — Firebase token verification & JWT session creation (`AuthContext.jsx` & `authController.js`)
2. ✅ **POST /api/doctors** — Creating a doctor profile & doctorCode generation (`DoctorDashboard.jsx` & `doctorController.js`)
3. 🎯 **POST /api/appointments** — Booking an appointment with consultation slot & fees (`BookingModal.jsx` & `appointmentController.js`)
4. ⏳ **POST /api/tokens/doctor/:doctorId** — Generating a live queue token for today (`TokenDisplay.jsx` & `tokenController.js`)
5. ⏳ **POST /api/uploads/signature** — Minting a secure Cloudinary upload signature (`PhotoUploader.jsx` & `uploadController.js`)

---

## Part 1 — Backend: Route Definition

File: [backend/routes/appointments.js](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/backend/routes/appointments.js)

```javascript
const express = require('express');
const asyncHandler = require('../middleware/asyncHandler');
const { protect } = require('../middleware/auth');
const { bookAppointment } = require('../controllers/appointmentController');

const router = express.Router();

router.post('/', protect, asyncHandler(bookAppointment));

module.exports = router;
```

### Line-by-Line Breakdown:
- `router.post('/', ...)`: Listens for incoming HTTP POST requests at `/api/appointments`.
- `protect`: Verifies that the client is an authenticated user. Both patients and doctors have active login tokens, but anyone making an appointment must have a valid identity.
- Notice there is **no `doctorOnly` middleware** here, because booking appointments is an action intended primarily for patients.

---

## Part 2 — Backend: Controller Logic

File: [backend/controllers/appointmentController.js](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/backend/controllers/appointmentController.js)

```javascript
const Appointment = require('../models/Appointment');
const Doctor = require('../models/Doctor');

// POST /api/appointments
async function bookAppointment(req, res) {
  const { doctorId, service, fee, date, timeSlot, notes } = req.body;

  // 1. Verify doctor existence
  const doctor = await Doctor.findById(doctorId);
  if (!doctor) return res.status(404).json({ message: 'Doctor not found.' });

  // 2. Create the appointment document
  const appointment = await Appointment.create({
    doctor: doctorId,
    patient: req.user.id,
    service,
    fee,
    date,
    timeSlot,
    notes,
  });

  res.status(201).json(appointment);
}
```

### Granular Line-by-Line Breakdown:
1. `const { doctorId, service, fee, date, timeSlot, notes } = req.body;`: Extracts the booking attributes passed from the client modal.
2. `const doctor = await Doctor.findById(doctorId)`: Validates that the targeted doctor actually exists in the database. If an invalid or deleted ID is passed, it halts with `404 Not Found`.
3. `patient: req.user.id`: **Security Best Practice**. Never trust the client to specify who is booking the appointment (`req.body.patientId`). By reading `req.user.id` from the verified JWT, the server guarantees the booking is linked to the authenticated user.
4. `Appointment.create({ ... })`: Creates the document with default status `'pending'`.
5. `res.status(201).json(appointment)`: Sends `201 Created` with the newly formed appointment object containing MongoDB `_id` and timestamps.

---

## Part 3 — Frontend: Controlled Modal Component

File: [frontend/src/components/BookingModal.jsx](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/frontend/src/components/BookingModal.jsx)

```javascript
export default function BookingModal({ doctor, onClose }) {
  const [service, setService] = useState(doctor.charges?.[0]?.service || '');
  const [date, setDate] = useState('');
  const [timeSlot, setTimeSlot] = useState('');
  const [error, setError] = useState('');
  const [success, setSuccess] = useState(false);

  // Automatically lookup fee associated with selected service
  const selectedCharge = doctor.charges?.find((c) => c.service === service);

  async function handleSubmit(e) {
    e.preventDefault();
    setError('');
    try {
      await api.post('/appointments', {
        doctorId: doctor._id,
        service,
        fee: selectedCharge?.fee || 0,
        date,
        timeSlot,
      });
      setSuccess(true);
    } catch (err) {
      setError(err.response?.data?.message || 'Could not book appointment.');
    }
  }

  return (
    <div className="fixed inset-0 z-[60] flex items-end justify-center bg-black/60 sm:items-center">
      <div className="bg-card p-[26px] rounded-t-[22px] sm:rounded-[22px]">
        {success ? (
          <div>
            <h3>Appointment requested!</h3>
            <p>Your appointment with {doctor.name} for {date} at {timeSlot} is pending confirmation.</p>
            <button onClick={onClose}>Done</button>
          </div>
        ) : (
          <form onSubmit={handleSubmit}>
            <h3>Book with {doctor.name}</h3>
            {error && <p className="text-danger">{error}</p>}
            {/* Service Selection */}
            <Field label="Service" as="select" value={service} onChange={(e) => setService(e.target.value)}>
              {doctor.charges?.map((c) => (
                <option key={c.service} value={c.service}>{c.service} — ₹{c.fee}</option>
              ))}
            </Field>
            {/* Date Input with Min=Today */}
            <Field label="Date" type="date" value={date} onChange={(e) => setDate(e.target.value)} min={new Date().toISOString().slice(0, 10)} required />
            {/* Time Slot Input */}
            <Field label="Preferred time" type="time" value={timeSlot} onChange={(e) => setTimeSlot(e.target.value)} required />
            <button type="submit">Confirm Booking</button>
          </form>
        )}
      </div>
    </div>
  );
}
```

### Line-by-Line Breakdown:
- `min={new Date().toISOString().slice(0, 10)}`: HTML5 constraint ensuring patients cannot select past dates in the calendar picker.
- `selectedCharge = doctor.charges?.find(...)`: Dynamically looks up the pricing configured by the doctor so the exact fee is passed with the payload.
- `await api.post('/appointments', { ... })`: Issues the HTTP POST request to Express.
- `setSuccess(true)`: Switches the modal view from the interactive form to the confirmation message.

---

## Part 4 — The Controlled Component Pattern

Notice that `BookingModal` receives `onClose` from its parent, [DoctorProfile.jsx](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/frontend/src/pages/DoctorProfile.jsx):

```javascript
{/* In DoctorProfile.jsx */}
{bookingOpen && (
  <BookingModal
    doctor={doctor}
    onClose={() => setBookingOpen(false)}
  />
)}
```

`BookingModal` does not maintain its own modal open/closed boolean state. Instead, the parent maintains `bookingOpen`. When the patient clicks the close button (`X`) or the "Done" confirmation button, `onClose()` calls `setBookingOpen(false)` in the parent. This architectural pattern keeps state predictable and eliminates desynchronization bugs.

---

## Part 5 — The Full End-to-End Data Flow

```
[Patient on DoctorProfile page]
       │
       ▼ 1. Clicks "Book Appointment" -> Opens <BookingModal />
       │
       ▼ 2. Selects service "Cardiology Consultation", date "2026-11-05", time "10:30"
       │
       ▼ 3. Submits form -> handleSubmit executes
       │
       ▼ 4. HTTP POST /api/appointments
            Headers:
              Authorization: Bearer <patient_token>
              Content-Type: application/json
            Body:
              {
                "doctorId": "651a1b...",
                "service": "Cardiology Consultation",
                "fee": 500,
                "date": "2026-11-05",
                "timeSlot": "10:30"
              }
       │
       ▼ 5. Express executes protect middleware -> verifies JWT and sets req.user
       │
       ▼ 6. bookAppointment controller:
            - Doctor.findById("651a1b...") -> Doctor verified
            - Appointment.create({ patient: req.user.id, ... })
            - Saved to MongoDB with status: "pending"
       │
       ▼ 7. Response: HTTP 201 Created
            Body: { _id: "652c4e...", status: "pending", date: "2026-11-05", ... }
       │
       ▼ 8. React setSuccess(true) renders green confirmation checkmark
```

---

## Part 6 — Quick Self-Test

1. **Why does the backend set `patient: req.user.id` instead of reading `req.body.patientId`?**  
   If the backend allowed `patientId` in the request body, a malicious user could tamper with the HTTP request and book appointments in the name of other patients. Extracting the ID from the cryptographically verified JWT guarantees authenticity.

2. **What HTTP status code is returned on successful appointment creation, and why?**  
   `201 Created`, because a new persistent record has been generated in the MongoDB `appointments` collection.

3. **Where can the patient see this newly created appointment later?**  
   On their personal dashboard at `/patient-dashboard` via `GET /api/appointments/my` (which we studied in Lesson 4 of GET requests).
