# Lesson 2: GET /api/doctors/:id (Fetching a Single Doctor's Complete Profile)

In **Lesson 1**, we saw how `GET /api/doctors` fetches the list of all doctors for the homepage directory.  
Now in **Lesson 2**, we explore what happens when a patient clicks **"View Profile"** on a doctor's card. We will track how the specific doctor's ID travels from the URL to Express, queries MongoDB by `_id`, and delivers the full profile data to `DoctorProfile.jsx`.

---

## Roadmap of GET Requests
1. ✅ **GET /api/doctors** — Fetching doctors list with search and filters (`Home.jsx`)
2. 🎯 **GET /api/doctors/:id** — Fetching a single doctor's complete profile (`DoctorProfile.jsx`)
3. ⏳ **GET /api/tokens/doctor/:doctorId** — Live token and queue status for a doctor (`TokenDisplay.jsx`)
4. ⏳ **GET /api/appointments/my** & **GET /api/tokens/my** — Patient's appointments and tokens (`PatientAppointments.jsx`)
5. ⏳ **GET /api/doctors/me/profile** — Logged-in doctor's profile (`DoctorDashboard.jsx`)
6. ⏳ **GET /api/tokens/doctor/:doctorId/queue** & **GET /api/appointments/doctor** — Doctor's live patient queue & appointments list (`DoctorDashboard.jsx`)

---

## Part 1 — Backend: Route Definition

**File:** `backend/routes/doctors.js`

```javascript
router.get('/', asyncHandler(getDoctors));
router.get('/me/profile', protect, doctorOnly, asyncHandler(getMyDoctorProfile));
router.get('/:id', asyncHandler(getDoctorById));
```

### Line-by-Line Breakdown:

- **`router.get('/:id', asyncHandler(getDoctorById));`**:
  - `/:id` is a **path parameter** (route parameter). The colon (`:`) tells Express that this segment of the URL is dynamic. Express will capture whatever is in that position and attach it to `req.params.id`.
  - For example, a request to `GET /api/doctors/64f8a12bc89e3a001a7d` sets `req.params.id = "64f8a12bc89e3a001a7d"`.
- **Why route ordering matters critically here**:
  - Notice `router.get('/me/profile', ...)` is placed **before** `router.get('/:id', ...)`.
  - Express evaluates routes from top to bottom. If `/:id` were declared first, a request to `/api/doctors/me/profile` might match `/:id` with `req.params.id = 'me'` and fail! Placing static sub-paths (`/me/profile`) before dynamic params (`/:id`) prevents route collisions.
- **No authentication needed (`protect`)**:
  - Just like the directory search, doctor profile details (clinic address, timings, charges, bio) are public information accessible to any patient or guest.

---

## Part 2 — Backend: Controller Logic

**File:** `backend/controllers/doctorController.js`

```javascript
// GET /api/doctors/:id
async function getDoctorById(req, res) {
  const doctor = await Doctor.findById(req.params.id);
  if (!doctor) return res.status(404).json({ message: 'Doctor not found.' });
  res.json(doctor);
}
```

### Line-by-Line Breakdown:

- **`async function getDoctorById(req, res)`**:
  - The controller function handling the GET request.
- **`const doctor = await Doctor.findById(req.params.id);`**:
  - `req.params.id` extracts the ID string from the URL path.
  - `Doctor.findById(...)` is a Mongoose query method that searches MongoDB's `doctors` collection where `_id` equals the given value.
  - Mongoose automatically converts the hex string into a MongoDB `ObjectId`.
- **`if (!doctor) return res.status(404).json({ message: 'Doctor not found.' });`**:
  - If no doctor with that ID exists in the database, `findById` returns `null`.
  - We immediately return an HTTP **404 Not Found** status with a descriptive JSON error message to stop execution.
- **`res.json(doctor);`**:
  - If found, sends HTTP status **200 OK** with the complete doctor object (including qualifications, bio, contact details, consultation charges, timings, clinic address, and coordinates).

---

## Part 3 — Frontend: Routing & Navigation

### 1. Route Registration in `App.jsx`

**File:** `frontend/src/App.jsx`

```jsx
<Route path="/doctor/:id" element={<DoctorProfile />} />
```

- When the browser URL matches `/doctor/<any_id>`, React Router mounts the `<DoctorProfile />` component.

### 2. Navigating from `DoctorCard.jsx`

**File:** `frontend/src/components/DoctorCard.jsx`

```jsx
<Link
  to={`/doctor/${doctor._id}`}
  className="flex items-center gap-[5px] text-[12.5px] font-bold text-brand after:absolute after:inset-0"
>
  View Profile <i className="fas fa-arrow-right"></i>
</Link>
```

- When the user clicks the card or the "View Profile" link, React Router performs a **client-side navigation** to `/doctor/${doctor._id}` without reloading the entire page.

---

## Part 4 — Frontend: Component Logic

**File:** `frontend/src/pages/DoctorProfile.jsx`

### 1. Reading the URL Param & State Initialization

```jsx
export default function DoctorProfile() {
  const { id } = useParams(); // reads the ":id" from the URL, e.g. /doctor/64f...
  const [doctor, setDoctor] = useState(null);
  const [error, setError] = useState('');
  const [showBooking, setShowBooking] = useState(false);
  const { user } = useAuth();
  const navigate = useNavigate();
```

| Variable / Hook | Initial Value | Purpose |
|---|---|---|
| `useParams()` | `{ id: "..." }` | React Router hook that extracts the `:id` parameter from the active route. |
| `doctor` | `null` | Holds the full doctor object once returned from the backend. |
| `error` | `''` | Holds error message if fetching fails or doctor is not found. |
| `showBooking` | `false` | Boolean toggle for opening the `<BookingModal />`. |
| `useAuth()` | `{ user }` | Checks if current visitor is logged in before booking. |
| `useNavigate()` | Function | For redirecting unauthenticated users to `/login`. |

---

### 2. The `useEffect` Hook to Fetch the Profile

```jsx
useEffect(() => {
  api.get(`/doctors/${id}`)
    .then((res) => setDoctor(res.data))
    .catch(() => setError('Could not load this doctor profile.'));
}, [id]);
```

- **`api.get(`/doctors/${id}`)`**: Sends an HTTP `GET` request to `/api/doctors/<id>`.
- **`.then((res) => setDoctor(res.data))`**: When the backend responds with 200 OK, `res.data` (the doctor object) is stored in the `doctor` state, triggering a re-render with the doctor's details.
- **`.catch(...)`**: If the doctor ID is invalid, 404, or the server is down, sets the error message.
- **Dependency array `[id]`**: Ensures if the user navigates directly from one doctor to another, the hook re-runs for the new ID.

---

### 3. Guard Clauses (Loading & Error States)

```jsx
if (error) return <Container><p className="py-10">{error}</p></Container>;
if (!doctor) return <Container><p className="py-10">Loading...</p></Container>;
```

- While the request is in flight, `doctor` is still `null`.
- These guard clauses prevent JavaScript runtime errors like `TypeError: Cannot read properties of null (reading 'name')` by rendering a clean loading or error placeholder until data arrives.

---

### 4. Derived Data & Subcomponents

Once `doctor` is loaded, the component computes additional data before rendering:

```jsx
// Google Maps directions deep-link using doctor's latitude & longitude
const mapsUrl = `https://www.google.com/maps/dir/?api=1&destination=${doctor.latitude},${doctor.longitude}`;

// Calculate the lowest consultation fee from the charges list
const minFee = doctor.charges?.length ? Math.min(...doctor.charges.map((c) => c.fee)) : null;
```

It also mounts the live token system component:
```jsx
<TokenDisplay doctorId={doctor._id} className="mt-0" />
```
*(This triggers our next GET request in Lesson 3!)*

---

## Part 5 — The Full End-to-End Data Flow

```
1. User clicks "View Profile" on DoctorCard (Home page)
        │
        ▼
2. React Router transitions route to `/doctor/64f8a12bc8...`
        │
        ▼
3. React mounts <DoctorProfile />
   - useParams() extracts id = "64f8a12bc8..."
   - State: doctor = null, error = ''
   - Guard clause executes: Renders "Loading..."
        │
        ▼
4. useEffect fires:
   Calls api.get('/doctors/64f8a12bc8...')
        │
        ▼
   ┌────────────────────────────────────────────────────────┐
   │ BROWSER SENDS HTTP GET                                  │
   │ GET /api/doctors/64f8a12bc8... HTTP/1.1                │
   │ Host: localhost:5173 (proxied to localhost:5000)       │
   └────────────────────────────────────────────────────────┘
        │
        ▼
5. Express routes the request:
   - Matches app.use('/api/doctors', doctorRoutes)
   - Matches router.get('/:id', asyncHandler(getDoctorById))
   - Express sets req.params.id = "64f8a12bc8..."
        │
        ▼
6. Controller executes:
   - Doctor.findById("64f8a12bc8...") runs in MongoDB
   - If document exists, returns doctor record
        │
        ▼
   ┌────────────────────────────────────────────────────────┐
   │ BACKEND SENDS HTTP RESPONSE                             │
   │ Status: 200 OK                                          │
   │ Content-Type: application/json                          │
   │ Body: { _id: "64f8...", name: "Dr. Sharma", ... }       │
   └────────────────────────────────────────────────────────┘
        │
        ▼
7. Frontend Axios receives response:
   - .then((res) => setDoctor(res.data)) updates state
        │
        ▼
8. DoctorProfile re-renders:
   - doctor is no longer null (guard clause skipped)
   - Header, Experience, Fees, Directions, Clinic details & Timings render
   - <TokenDisplay doctorId={doctor._id} /> mounts
        │
        ▼
9. Patient sees full profile screen with live token status ✅
```

---

## Part 6 — Quick Self-Test

1. **Why must `router.get('/me/profile', ...)` appear before `router.get('/:id', ...)` in `doctors.js`?**  
   *Because Express matches routes in order. If `/:id` were first, Express would treat `'me'` as the `:id` parameter and route it to `getDoctorById` instead of `getMyDoctorProfile`.*

2. **How does `DoctorProfile.jsx` access the doctor ID from the browser address bar?**  
   *Using the React Router `useParams()` hook, which returns an object containing key-value pairs of dynamic URL segments defined in `<Route path="/doctor/:id" />`.*

3. **Why do we need `if (!doctor) return <Container>Loading...</Container>;` before accessing `doctor.name`?**  
   *Because HTTP requests are asynchronous. On initial render, `doctor` is `null`. Attempting to access `doctor.name` before data arrives would crash the app with a `TypeError`.*
