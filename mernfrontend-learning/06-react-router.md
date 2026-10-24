# Module 06: Routing & Single Page Applications (`react-router-dom` v6)

---

## 1. 💡 Mental Model: Client-Side Routing vs Traditional Web

In a traditional multi-page website:
1. You click a link `<a href="/doctor/123">`.
2. The browser sends an HTTP GET request to the web server.
3. The server builds an HTML page and sends it back.
4. The browser discards the current page (blank white screen flash) and re-parses all HTML, CSS, and JS scripts.

In a **React Single-Page Application (SPA)**:
1. You load the JavaScript bundle once (`index.html`).
2. You click a `<Link to="/doctor/123">`.
3. `react-router-dom` intercepts the click, calls `window.history.pushState()`, and changes the browser URL.
4. **No network request is made for HTML**. React simply swaps out the active component on the page in milliseconds!

---

## 2. 🗺️ Route Configuration in `App.jsx`

Here is our application's route tree:

```jsx
// frontend/src/App.jsx
import { Routes, Route } from 'react-router-dom';
import Navbar from './components/Navbar';
import Home from './pages/Home';
import Login from './pages/Login';
import Register from './pages/Register';
import DoctorProfile from './pages/DoctorProfile';
import PatientAppointments from './pages/PatientAppointments';
import DoctorDashboard from './pages/DoctorDashboard';

export default function App() {
  return (
    <>
      {/* Navbar sits OUTSIDE <Routes> so it stays mounted on every page */}
      <Navbar />
      <main>
        <Routes>
          <Route path="/" element={<Home />} />
          <Route path="/login" element={<Login />} />
          <Route path="/register" element={<Register />} />
          
          {/* Dynamic route with URL parameter */}
          <Route path="/doctor/:id" element={<DoctorProfile />} />
          
          {/* Protected Views */}
          <Route path="/my-appointments" element={<PatientAppointments />} />
          <Route path="/doctor-dashboard" element={<DoctorDashboard />} />
        </Routes>
      </main>
    </>
  );
}
```

---

## 3. 🔍 Dynamic Routes & Hooks Deep-Dive

### 1. `useParams()`: Reading URL Parameters
In `DoctorProfile.jsx`, we read the doctor's MongoDB ID from the URL:
```jsx
// URL: /doctor/6652e9f45d8b
import { useParams } from 'react-router-dom';

export default function DoctorProfile() {
  const { id } = useParams(); // id === "6652e9f45d8b"

  useEffect(() => {
    api.get(`/doctors/${id}`).then(res => setDoctor(res.data));
  }, [id]);
}
```

### 2. `useSearchParams()`: Reading & Setting Query Strings
In `DoctorDashboard.jsx`, we detect if this is a first-time setup via query params (`/doctor-dashboard?welcome=1`):
```jsx
import { useSearchParams } from 'react-router-dom';

export default function DoctorDashboard() {
  const [searchParams, setSearchParams] = useSearchParams();
  const showWelcome = searchParams.get('welcome') === '1';

  // Can also programmatically update query params:
  // setSearchParams({ tab: 'queue' });
}
```

### 3. `useNavigate()`: Programmatic Navigation
Used when navigating after an async action (like login or form submit):
```jsx
import { useNavigate } from 'react-router-dom';

const navigate = useNavigate();

// 1. Standard navigation:
navigate('/my-appointments');

// 2. Navigation with history replacement (prevents pressing "Back" to return to login):
navigate('/doctor-dashboard', { replace: true });

// 3. Go back one step:
navigate(-1);
```

---

## 4. 🛡️ Implementing Production Route Guards (`ProtectedRoute`)

In medical software, a patient should never view a doctor's consultation queue, and an unauthenticated user shouldn't view appointments.

Here is how you build a reusable **Route Guard**:

```jsx
// frontend/src/components/ProtectedRoute.jsx
import { Navigate, useLocation } from 'react-router-dom';
import { useAuth } from '../context/AuthContext';

export default function ProtectedRoute({ allowedRoles, children }) {
  const { user } = useAuth();
  const location = useLocation();

  if (!user) {
    // Redirect to login, but remember where they tried to go!
    return <Navigate to="/login" state={{ from: location }} replace />;
  }

  if (allowedRoles && !allowedRoles.includes(user.role)) {
    // If patient tries to access doctor-dashboard:
    return <Navigate to="/" replace />;
  }

  return children;
}
```

#### Wrapping Routes in `App.jsx`:
```jsx
<Route 
  path="/doctor-dashboard" 
  element={
    <ProtectedRoute allowedRoles={['doctor']}>
      <DoctorDashboard />
    </ProtectedRoute>
  } 
/>
```

---

## 5. ⚠️ Real-World Production War Story: The "404 On Page Refresh" Disaster

### The Scenario:
A team builds a React SPA. Everything works perfectly in local development with Vite (`http://localhost:5173/doctor/123`).
They deploy the production build to an Nginx server, AWS S3, or Apache.
A user visits `https://myclinic.com/`, clicks on Dr. Sharma (`https://myclinic.com/doctor/123`), and everything works.
**Then the user hits REFRESH (F5).**
$\rightarrow$ **`404 Not Found: The requested URL was not found on this server.`**

### Why It Happens:
In local dev, Vite intercepts all requests and redirects them to `index.html`.
On a production web server (Nginx/Apache), when the user requests `/doctor/123`, the server looks on its hard drive for a file named `/doctor/123.html`. Because that file doesn't exist (only `index.html` exists!), it returns a 404!

### The Senior Engineer Fix:
Configure the web server to rewrite all fallback requests to `index.html`:
```nginx
# Nginx Configuration:
location / {
  try_files $uri $uri/ /index.html;
}
```
Or for Netlify (`_redirects` file):
```
/*    /index.html   200
```
Or for Vercel (`vercel.json`):
```json
{
  "rewrites": [{ "source": "/(.*)", "destination": "/index.html" }]
}
```

---

## 6. 🎯 Interview Question Bank: React Router

### Q1: What is the difference between `<Link>` and `<a href>`?
> **Answer**:
> - `<a href>` triggers a full browser page refresh: closes the current document, drops in-memory React state, sends a new request to the server, and reloads all script bundles.
> - `<Link>` intercepts the click event using `e.preventDefault()`, updates the browser URL using the HTML5 `history.pushState` API, and asks React Router to render the matching component without reloading the page.

### Q2: What is the difference between `useNavigate()` and `<Navigate />`?
> **Answer**:
> - `useNavigate()` is an imperative hook used inside functions/callbacks (e.g., inside an `onSubmit` or `onClick` handler).
> - `<Navigate />` is a declarative component used inside the JSX render return tree (e.g., returning `<Navigate to="/login" />` from a Protected Route when a condition fails).

### Q3: Why is `{ replace: true }` important during authentication redirects?
> **Answer**:
> If a user logs in and you use standard navigation (`navigate('/dashboard')`), clicking the browser "Back" button pushes them right back to `/login`.
> Using `{ replace: true }` replaces the `/login` entry in the browser's history stack with `/dashboard`, so clicking "Back" returns them to the page they were on *before* they went to login.

---

## 7. 💻 Hands-On Mastery Exercise

Implement a `404 Not Found` catch-all route in `App.jsx` using `<Route path="*" element={<NotFound />} />` with a friendly "Return to Home" button.
