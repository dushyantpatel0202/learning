# Module 04: Advanced Hooks (`useRef`, `useMemo`, `useCallback` & Custom Hooks)

---

## 1. 💡 Mental Model: Beyond Basic State

While `useState` and `useEffect` handle 80% of everyday React tasks, high-performance applications require fine-grained control over:
1. **DOM elements and mutable values that don't trigger re-renders** (`useRef`).
2. **Expensive calculations that shouldn't re-run every time** (`useMemo`).
3. **Stable function references passed to optimized child components** (`useCallback`).
4. **Clean, reusable business logic abstracted from UI components** (Custom Hooks).

---

## 2. 📌 `useRef` Deep-Dive

`useRef` returns a mutable object: `{ current: initialValue }`.
It has two distinct superpowers:
1. **Holding a direct reference to a real DOM element** (like `document.getElementById`).
2. **Persisting a mutable value across renders WITHOUT triggering a re-render** when it changes.

### Use-Case 1: Direct DOM Manipulation (Focusing an Input)
```jsx
import { useRef, useEffect } from 'react';

export function SearchInput() {
  const inputRef = useRef(null);

  useEffect(() => {
    // Automatically focus the input when page mounts
    inputRef.current?.focus();
  }, []);

  return <input ref={inputRef} placeholder="Search doctors..." />;
}
```

### Use-Case 2: Storing Timers / Intervals Without Re-rendering
```jsx
function LiveQueueTimer() {
  const [seconds, setSeconds] = useState(0);
  const timerRef = useRef(null); // Keeps ID across renders

  function start() {
    if (timerRef.current !== null) return;
    timerRef.current = setInterval(() => {
      setSeconds(prev => prev + 1);
    }, 1000);
  }

  function stop() {
    clearInterval(timerRef.current);
    timerRef.current = null; // Doesn't trigger a re-render!
  }

  return <div>{seconds}s <button onClick={stop}>Pause</button></div>;
}
```

---

## 3. 🧠 `useMemo`: Caching Expensive Computations

`useMemo` caches the **result** of a calculation between renders:
```javascript
const memoizedValue = useMemo(() => computeExpensiveValue(a, b), [a, b]);
```

### Real Project Case: Doctor Distance Calculation
In `frontend/src/pages/Home.jsx`, when a patient clicks "Near Me", we calculate the Haversine distance between their coordinates and 200+ doctors:

```jsx
import { useMemo } from 'react';
import { distanceInKm } from '../utils/distance';

export function DoctorDirectory({ doctors, userCoords, filterSpecialization }) {
  // Recalculates ONLY when 'doctors', 'userCoords', or 'filterSpecialization' changes.
  // Re-rendering because of an unrelated state change (e.g., dark mode toggle) SKIPS this loop!
  const sortedAndFilteredDoctors = useMemo(() => {
    console.log("⚡ Calculating distances and sorting...");
    
    return doctors
      .filter(doc => !filterSpecialization || doc.specialization === filterSpecialization)
      .map(doc => ({
        ...doc,
        distance: userCoords 
          ? distanceInKm(userCoords.lat, userCoords.lng, doc.latitude, doc.longitude) 
          : null
      }))
      .sort((a, b) => (a.distance ?? 0) - (b.distance ?? 0));
  }, [doctors, userCoords, filterSpecialization]);

  return (
    <div>
      {sortedAndFilteredDoctors.map(doc => (
        <DoctorCard key={doc._id} doctor={doc} />
      ))}
    </div>
  );
}
```

> [!TIP]
> **When NOT to use `useMemo`**:
> Do not memoize cheap operations (like `const active = doctors.length > 0`). The overhead of creating the array dependency, storing the closure, and comparing dependencies costs more CPU cycles than the calculation itself!

---

## 4. 🔗 `useCallback`: Preserving Function Identity

In JavaScript: `() => {} !== () => {}`.
Every time a component re-renders, all functions declared inside it are **re-created with brand new memory addresses**.

```jsx
// Without useCallback:
// Parent renders -> new reference of 'handleCancel' created -> Child re-renders even if wrapped in React.memo!
const handleCancel = (appointmentId) => {
  api.delete(`/appointments/${appointmentId}`);
};

// With useCallback:
// Same function instance preserved across renders as long as dependencies don't change:
const handleCancel = useCallback((appointmentId) => {
  api.delete(`/appointments/${appointmentId}`);
}, []);
```

---

## 5. 🛠️ Building Production Custom Hooks

A custom hook is simply a JavaScript function whose name starts with `use` and that can call other React hooks.

### Production Custom Hook: `useDebounce`
When a user searches for a doctor, we don't want to hit the backend API on every single keystroke. We wait until they pause typing for 400ms:

```jsx
// frontend/src/hooks/useDebounce.js
import { useState, useEffect } from 'react';

export function useDebounce(value, delay = 400) {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const handler = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    // Cancel the timer if 'value' changes before delay expires (typing continues)
    return () => {
      clearTimeout(handler);
    };
  }, [value, delay]);

  return debouncedValue;
}
```

#### How to use it in `Home.jsx`:
```jsx
export default function Home() {
  const [search, setSearch] = useState('');
  const debouncedSearch = useDebounce(search, 400);

  // Automatically fires only when the user finishes typing!
  useEffect(() => {
    fetchDoctors({ search: debouncedSearch });
  }, [debouncedSearch]);

  return (
    <input 
      value={search} 
      onChange={(e) => setSearch(e.target.value)} 
      placeholder="Search doctors..." 
    />
  );
}
```

---

## 6. ⚠️ Real-World Production War Story: The Stale Closure Trap

### The Bug:
A hospital queue monitor updated the next patient token:
```jsx
function QueueMonitor() {
  const [queue, setQueue] = useState([]);

  // ❌ STALE CLOSURE BUG:
  const processNext = useCallback(() => {
    if (queue.length === 0) return;
    const nextPatient = queue[0];
    notifyPatient(nextPatient);
    setQueue(queue.slice(1)); // 'queue' here is FROZEN from the first render!
  }, []); // Empty dependency array!
}
```
**What happened in production**:
When `processNext` was called repeatedly, it always used the initial empty array `queue = []`. The queue refused to advance!

**The Fix**:
Either add `queue` to the dependency array `[queue]`, or use the functional state updater which never goes stale:
```jsx
const processNext = useCallback(() => {
  setQueue(prevQueue => {
    if (prevQueue.length === 0) return prevQueue;
    const [nextPatient, ...remaining] = prevQueue;
    notifyPatient(nextPatient);
    return remaining;
  });
}, []); // Safe to keep empty because we don't read 'queue' directly!
```

---

## 7. 🎯 Interview Question Bank: Advanced Hooks

### Q1: What is the core difference between `useMemo` and `useCallback`?
> **Answer**:
> - `useMemo` caches the **result of executing a function** (a calculated number, filtered array, transformed object).
> - `useCallback` caches the **function definition itself** (the function reference) to prevent child components from re-rendering due to broken referential equality.
> In fact: `useCallback(fn, deps)` is equivalent to `useMemo(() => fn, deps)`.

### Q2: Why does modifying `ref.current` NOT trigger a component re-render?
> **Answer**:
> `useRef` is a plain JavaScript object whose reference never changes between renders. React's reconciliation engine only triggers re-renders in response to state changes (`useState`, `useReducer`) or prop changes. Mutating an internal property of a reference object bypasses React's render phase entirely.

### Q3: When should you wrap a component in `React.memo`?
> **Answer**:
> Only when:
> 1. The component renders often with the exact same props.
> 2. The component is heavy/computationally expensive to re-render.
> Do NOT wrap every component indiscriminately: `React.memo` adds a shallow prop comparison step on every render, which adds overhead if props change on almost every render anyway.

### Q4: Code Output & Reference Check
**Will Child re-render when the button is clicked?**
```jsx
const Child = React.memo(({ config, onClick }) => {
  console.log("Child rendered");
  return <button onClick={onClick}>{config.title}</button>;
});

function Parent() {
  const [count, setCount] = useState(0);

  const handleClick = () => console.log("clicked");
  const config = { title: "Submit" };

  return (
    <div>
      <button onClick={() => setCount(count + 1)}>Increment ({count})</button>
      <Child config={config} onClick={handleClick} />
    </div>
  );
}
```
> **Answer**:
> **Yes, Child WILL re-render every time the button is clicked!**
> Even though `Child` is wrapped in `React.memo`, both `handleClick` and `config` are re-created as brand new references on every `Parent` re-render:
> `{} !== {}` and `() => {} !== () => {}`.
> **To fix it**: Wrap `config` in `useMemo` (or move outside the component) and wrap `handleClick` in `useCallback`.

---

## 8. 💻 Hands-On Mastery Exercise

Implement a custom hook `useLocalStorage(key, initialValue)` that:
1. Reads initial value from `localStorage`.
2. Syncs any state updates automatically back into `localStorage`.
3. Handles `JSON.stringify` and `JSON.parse` errors safely.
