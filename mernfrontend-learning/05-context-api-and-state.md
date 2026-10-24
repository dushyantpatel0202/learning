# Module 05: Context API & Global State Management

---

## 1. 💡 Mental Model: The Need for Global State

In a real-world MERN application, certain data must be accessible anywhere in the component tree without manually threading props through 10 levels:
- **Authenticated User Session** (`user`, `token`, `role`)
- **Theme / Dark Mode**
- **Active Notifications / Toast Alerts**
- **Cart or Live Appointment Queue Status**

React provides the **Context API** as a native solution for broadcasting data to any subtree.

```
                  <AuthProvider> (Holds state: user, login, logout)
                        │
         ┌──────────────┴──────────────┐
         ▼                             ▼
     <Navbar>                      <Routes>
  (reads user.name,            ┌───────┴───────┐
   shows "Logout")             ▼               ▼
                           <Home>      <DoctorDashboard>
                                     (reads user.role, 
                                      guards page)
```

---

## 2. 🏗️ The 3-Step Pattern of React Context

### Step 1: Create the Context
```javascript
import { createContext } from 'react';

// Pass null or a fallback default value
export const AuthContext = createContext(null);
```

### Step 2: Create the Provider Component
The Provider holds the state and supplies it to descendants via the `value` prop:
```jsx
export function AuthProvider({ children }) {
  const [user, setUser] = useState(null);

  const login = (userData) => setUser(userData);
  const logout = () => setUser(null);

  return (
    <AuthContext.Provider value={{ user, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
}
```

### Step 3: Consume with a Custom Hook
Always expose a custom hook (like `useAuth`) instead of forcing components to import `useContext` and `AuthContext` separately:
```javascript
import { useContext } from 'react';

export function useAuth() {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within an AuthProvider');
  }
  return context;
}
```

---

## 3. 🏥 Real Project Code Walkthrough: `frontend/src/context/AuthContext.jsx`

Let's look at how our real Doctor Appointment app manages authentication across the entire frontend:

```jsx
// frontend/src/context/AuthContext.jsx
import { createContext, useContext, useEffect, useState } from 'react';
import { useNavigate } from 'react-router-dom';
import { signOut } from 'firebase/auth';
import api from '../utils/api';
import { auth } from '../firebase';

const AuthContext = createContext(null);

function readSavedUser() {
  try {
    return JSON.parse(localStorage.getItem('user')) || null;
  } catch {
    return null;
  }
}

export function AuthProvider({ children }) {
  const [user, setUser] = useState(readSavedUser);
  const navigate = useNavigate();

  async function completeSignIn(profile = {}) {
    const firebaseUser = auth?.currentUser;
    if (!firebaseUser) throw new Error('Please sign in first.');

    // Fetch fresh token so verified status is updated
    const idToken = await firebaseUser.getIdToken(true);
    let pending = JSON.parse(sessionStorage.getItem('pendingProfile') || '{}');

    // Exchange Firebase token for our backend JWT session:
    const { data } = await api.post('/auth/session', { idToken, ...pending, ...profile });
    if (data.needsProfile) return data;

    localStorage.setItem('token', data.token);
    localStorage.setItem('user', JSON.stringify(data.user));
    setUser(data.user);
    return data;
  }

  async function logout() {
    try {
      if (auth) await signOut(auth);
    } catch {}
    localStorage.removeItem('token');
    localStorage.removeItem('user');
    setUser(null);
  }

  return (
    <AuthContext.Provider value={{ user, completeSignIn, finishSignIn, logout }}>
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth() {
  return useContext(AuthContext);
}
```

### Why this is production-grade:
1. **Lazy state initialization (`readSavedUser`)**: On page refresh, React immediately initializes `user` from `localStorage`, preventing an ugly "flicker" where the user looks logged out for half a second.
2. **Cohesive business logic**: Login flow, token exchange, profile completion routing, and logout teardown are centralized in one single file.
3. **Decoupled consumption**: Any page (e.g. `DoctorDashboard.jsx` or `DoctorProfile.jsx`) simply calls:
   ```javascript
   const { user, logout } = useAuth();
   ```

---

## 4. ⚠️ Real-World Production War Story: The "Unnecessary Re-render Avalanche"

### The Bug:
In an enterprise healthcare dashboard, the developer put both the `activeTheme` ("light" / "dark") and the `livePatientsQueue` (updates 5 times per second) inside the same Context:

```jsx
<GlobalContext.Provider value={{ theme, setTheme, queue, setQueue }}>
  <App />
</GlobalContext.Provider>
```

**What Went Wrong in Production**:
Whenever a new patient checked in or an appointment was called, `queue` updated $\rightarrow$ the Provider's `value` object reference changed $\rightarrow$ **EVERY component consuming `useGlobalContext()` re-rendered**, including the static sidebar, settings forms, and footer! The app became sluggish and unresponsive on low-end devices.

### Senior Engineering Solution:
1. **Split Contexts**: Keep frequently changing state separate from static/rarely changing state (`ThemeContext` vs `QueueContext`).
2. **Memoize the Provider `value`**:
   ```jsx
   const value = useMemo(() => ({ user, login, logout }), [user]);
   return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
   ```
3. **Use Zustand or Redux Toolkit** when dealing with high-frequency, fine-grained state updates with slice selectors.

---

## 5. 🎯 Interview Question Bank: Context API & State Management

### Q1: When would you choose Context API vs Redux Toolkit vs Zustand?
> **Answer**:
> - **React Context API**: Best for low-frequency, global app settings (Auth session, theme, localization/language, global modal alerts). Native to React, zero bundle size overhead.
> - **Zustand**: Best for medium-to-large fullstack applications needing high performance with minimal boilerplate. Supports fine-grained component subscription (components only re-render if the exact slice of state they select changes).
> - **Redux Toolkit (RTK)**: Best for complex enterprise apps with large engineering teams requiring strict state machines, predictable serialization, time-travel debugging, and standardized middleware.

### Q2: Why does creating an object inline in `value={{ user, logout }}` trigger re-renders?
> **Answer**:
> Because on every render of the Provider component, `{ user, logout }` creates a **brand new object reference in memory** (`{} !== {}`). Every consumer component detecting the new reference will be forced to re-render, even if `user` itself did not change.
> Wrapping the value in `useMemo` preserves referential equality across Provider re-renders.

### Q3: How do you guard against calling a custom hook outside its Provider?
> **Answer**:
> Check if `useContext(MyContext)` returns `null` or `undefined`, and throw an explicit descriptive Error:
> ```javascript
> export function useAuth() {
>   const ctx = useContext(AuthContext);
>   if (!ctx) throw new Error("useAuth must be wrapped inside <AuthProvider />");
>   return ctx;
> }
> ```
> This prevents cryptic runtime errors like `Cannot read properties of undefined (reading 'user')`.

### Q4: Code Output & Architecture Question
**If Component C updates the context, which components will re-render?**
```
     <App>
   ┌───┴───┐
  <A>     <B>
   │       │
  <C>     <D> (Consumes AuthContext)
```
> **Answer**:
> If `<AuthProvider>` re-renders to provide the new context value:
> 1. `<AuthProvider>` re-renders.
> 2. All consumers (like `<D>`) will re-render.
> 3. `<A>` and `<B>` will only re-render if they are direct children inside the provider and not passed via the `children` prop (or if they are not wrapped in `React.memo`).
> Because `<App>` wraps children via `<AuthProvider>{children}</AuthProvider>`, React skips re-rendering `<A>` and `<B>` if their props didn't change!

---

## 6. 💻 Hands-On Mastery Exercise

Look at `frontend/src/context/AuthContext.jsx`.
Memoize the `value` object passed to `AuthContext.Provider` using `useMemo` so that only changes to `user` trigger consumer updates.
