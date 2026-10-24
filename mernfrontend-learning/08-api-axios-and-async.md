# Module 08: API Architecture, Axios Interceptors & Async State

---

## 1. 💡 Mental Model: Decoupled API Layer

In novice React code, `fetch('http://localhost:5000/api/doctors')` is scattered across 20 different components.
If the API URL changes, or token authentication is added, or an interceptor is needed, 20 files must be updated.

Professional full-stack architecture uses a **Centralized API Client Layer**:
- Base URL configuration.
- Dev server proxy configuration to prevent CORS issues.
- **Request Interceptors**: Inject authentication headers transparently.
- **Response Interceptors**: Catch global 401 Unauthorized errors (token expired) and trigger auto-logout.
- Strict UI state transitions: `Loading -> Success | Error | Empty`.

---

## 2. 🏥 Real Project Code Walkthrough: `frontend/src/utils/api.js`

Here is our application's shared Axios client:

```javascript
// frontend/src/utils/api.js
import axios from 'axios';

// 1. Create a custom Axios instance with a relative baseURL:
const api = axios.create({
  baseURL: '/api', 
});

// 2. REQUEST INTERCEPTOR:
// Runs before EVERY outgoing HTTP request.
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

export default api;
```

### Enhancing with a RESPONSE INTERCEPTOR (Senior Interview Pattern)
What happens when a doctor's JWT token expires while they are in the dashboard?
Without a response interceptor, the API returns a `401 Unauthorized`, and the user is left looking at broken buttons.

Here is how you handle it globally:
```javascript
// Global 401 auto-logout handler:
api.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      // Clear expired credentials
      localStorage.removeItem('token');
      localStorage.removeItem('user');
      // Redirect to login
      window.location.href = '/login?expired=true';
    }
    return Promise.reject(error);
  }
);
```

---

## 3. 🌐 Demystifying CORS & The Vite Dev Proxy

### What is CORS (Cross-Origin Resource Sharing)?
Browsers enforce the **Same-Origin Policy**.
An origin is defined by: `Protocol + Domain + Port`.
- React runs at: `http://localhost:5173`
- Node/Express runs at: `http://localhost:5000`

Because the **ports differ** (`5173` vs `5000`), the browser will block requests unless the backend explicitly sets CORS headers.

### The Solution: Vite Reverse Proxy
In `frontend/vite.config.js`:
```javascript
export default defineConfig({
  plugins: [react(), tailwindcss()],
  server: {
    proxy: {
      '/api': {
        target: 'http://localhost:5000',
        changeOrigin: true,
      },
    },
  },
});
```

#### How it works:
1. React makes a request to `/api/doctors` on port `5173`.
2. The browser considers this **Same-Origin** (port `5173` talking to port `5173`) $\rightarrow$ **No CORS check triggered!**
3. The Vite development server silently forwards the request in the background to `http://localhost:5000/api/doctors`.
4. Servers talking to servers are not subject to browser CORS policies!

---

## 4. 📊 The 4 States of Async UI

Every production UI that fetches data must handle 4 distinct states:
1. **Loading State**: Show a skeleton loader or spinner.
2. **Error State**: Show a friendly error alert with a retry button.
3. **Empty State**: Show a prompt when the array returns 0 items (e.g., "No doctors found matching that search").
4. **Success State**: Render the actual data.

### Standard Production Component Structure:
```jsx
export function DoctorList() {
  const [doctors, setDoctors] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState('');

  // 1. Loading
  if (loading) {
    return <div className="spinner">Loading doctors...</div>;
  }

  // 2. Error
  if (error) {
    return (
      <div className="error-box">
        <p>{error}</p>
        <button onClick={fetchDoctors}>Try Again</button>
      </div>
    );
  }

  // 3. Empty State
  if (doctors.length === 0) {
    return (
      <div className="empty-box">
        <p>No doctors found matching your criteria.</p>
      </div>
    );
  }

  // 4. Success State
  return (
    <div className="grid">
      {doctors.map(doc => <DoctorCard key={doc._id} doctor={doc} />)}
    </div>
  );
}
```

---

## 5. ⚠️ Real-World Production War Story: The Infinite 401 Refresh Loop

### The Bug:
A company implemented a refresh-token interceptor in Axios. Whenever a request failed with 401, the interceptor called `/api/auth/refresh-token` to get a new access token, then retried the original request.
When the refresh token expired as well, the refresh-token endpoint returned a 401.
The response interceptor caught that 401, called `/api/auth/refresh-token` again, which failed with 401...
$\rightarrow$ **The browser blasted 10,000 HTTP requests per minute, DDOS-ing their own authentication server!**

### The Senior Engineer Fix:
Mark the request with an internal retry flag (`_retry = true`) so the interceptor only retries **once**:
```javascript
api.interceptors.response.use(
  (res) => res,
  async (error) => {
    const originalRequest = error.config;
    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true; // Mark as retried!
      try {
        const { data } = await axios.post('/api/auth/refresh-token');
        localStorage.setItem('token', data.token);
        originalRequest.headers.Authorization = `Bearer ${data.token}`;
        return api(originalRequest); // Retry once
      } catch (refreshErr) {
        // Refresh token failed -> Force full logout!
        localStorage.clear();
        window.location.href = '/login';
        return Promise.reject(refreshErr);
      }
    }
    return Promise.reject(error);
  }
);
```

---

## 6. 🎯 Interview Question Bank: API & Axios

### Q1: What is the difference between `fetch` and `axios`?
> **Answer**:
> | Feature | `fetch` | `axios` |
> | :--- | :--- | :--- |
> | JSON Parsing | Requires manual `await res.json()` | Automatically parses JSON response |
> | HTTP Error Handling | Does NOT reject on 404/500 (only on network drop) | Rejects promises on any non-2xx status code |
> | Interceptors | No built-in interceptors | Built-in request & response interceptors |
> | Request Cancellation | Requires manual `AbortController` | First-class `CancelToken` and `AbortSignal` support |
> | Progress Monitoring | Complex readable streams | Built-in `onUploadProgress` / `onDownloadProgress` |

### Q2: Why does `fetch` NOT reject on a 404 or 500 status code?
> **Answer**:
> By specification, `fetch()` only rejects a Promise when a **network error occurs** (e.g. computer disconnected from internet, DNS resolution fails, or CORS blocks the request). A 404 or 500 is still considered a successful HTTP communication exchange from the browser's networking layer perspective. Developers must manually check `if (!res.ok)`.

### Q3: What is the purpose of the `Authorization: Bearer <token>` header?
> **Answer**:
> The `Bearer` authentication scheme is an HTTP standard (RFC 6750) used with OAuth 2.0 and JWTs. It informs the backend server that the bearer (holder) of this token is authorized to perform actions on behalf of the user identified inside the encrypted or signed payload.

---

## 7. 💻 Hands-On Mastery Exercise

Look at `frontend/src/utils/api.js`. Add an Axios response interceptor that logs the response time (in milliseconds) of every API call in development to identify slow backend database queries.
