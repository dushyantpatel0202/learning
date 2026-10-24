# Module 12: Frontend Machine Coding Drills & Interview Puzzles

---

## 1. 🎯 Machine Coding Challenge 1: Debounced Search Autocomplete

**Interviewer Prompt**:
> *"Build an accessible search autocomplete component for our doctor directory. As the user types, it should debounce network requests by 300ms, display loading state, allow keyboard navigation (Arrow Up, Arrow Down, Enter to select), and handle outside click dismissal."*

### Production Solution:
```jsx
import { useState, useEffect, useRef } from 'react';

export function DoctorAutocomplete({ onSelectDoctor }) {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [loading, setLoading] = useState(false);
  const [isOpen, setIsOpen] = useState(false);
  const [activeIndex, setActiveIndex] = useState(-1);
  const containerRef = useRef(null);

  // Debounced API search effect
  useEffect(() => {
    if (!query.trim()) {
      setResults([]);
      setIsOpen(false);
      return;
    }

    const timer = setTimeout(async () => {
      setLoading(true);
      try {
        const res = await fetch(`/api/doctors?search=${encodeURIComponent(query)}`);
        const data = await res.json();
        setResults(data);
        setIsOpen(true);
        setActiveIndex(-1);
      } catch (err) {
        console.error(err);
      } finally {
        setLoading(false);
      }
    }, 300);

    return () => clearTimeout(timer);
  }, [query]);

  // Handle outside click to dismiss dropdown
  useEffect(() => {
    function handleClickOutside(e) {
      if (containerRef.current && !containerRef.current.contains(e.target)) {
        setIsOpen(false);
      }
    }
    document.addEventListener('mousedown', handleClickOutside);
    return () => document.removeEventListener('mousedown', handleClickOutside);
  }, []);

  // Keyboard navigation handler
  function handleKeyDown(e) {
    if (!isOpen) return;

    if (e.key === 'ArrowDown') {
      e.preventDefault();
      setActiveIndex(prev => (prev < results.length - 1 ? prev + 1 : 0));
    } else if (e.key === 'ArrowUp') {
      e.preventDefault();
      setActiveIndex(prev => (prev > 0 ? prev - 1 : results.length - 1));
    } else if (e.key === 'Enter') {
      e.preventDefault();
      if (activeIndex >= 0 && results[activeIndex]) {
        handleSelect(results[activeIndex]);
      }
    } else if (e.key === 'Escape') {
      setIsOpen(false);
    }
  }

  function handleSelect(doc) {
    setQuery(doc.name);
    setIsOpen(false);
    onSelectDoctor(doc);
  }

  return (
    <div ref={containerRef} className="relative w-full max-w-md">
      <input
        type="text"
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        onKeyDown={handleKeyDown}
        placeholder="Search doctor by name or specialty..."
        className="w-full rounded-field border border-line bg-card px-4 py-2.5 text-main focus:outline-brand"
      />
      {loading && <span className="absolute right-3 top-3 text-xs text-mute">Loading...</span>}

      {isOpen && results.length > 0 && (
        <ul className="absolute z-50 mt-1 max-h-60 w-full overflow-auto rounded-card border border-line bg-card shadow-lg">
          {results.map((doc, idx) => (
            <li
              key={doc._id}
              onClick={() => handleSelect(doc)}
              className={`cursor-pointer px-4 py-2.5 text-sm transition-colors ${
                idx === activeIndex ? 'bg-brand/15 text-brand font-semibold' : 'text-sub hover:bg-soft'
              }`}
            >
              <div>{doc.name}</div>
              <div className="text-xs text-mute">{doc.specialization}</div>
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

---

## 2. 🧩 Output Puzzles Commonly Asked in Senior Rounds

### Puzzle 1: Batching & State Reference
**What prints to console?**
```jsx
function Quiz() {
  const [val, setVal] = useState(1);

  const handleClick = () => {
    setVal(val + 1);
    setVal(val + 1);
    setVal(prev => prev + 2);
  };

  console.log("Render:", val);
  return <button onClick={handleClick}>Run</button>;
}
```
**Answer**:
When button is clicked:
1. `setVal(val + 1)` schedules update with $1 + 1 = 2$.
2. `setVal(val + 1)` schedules update with $1 + 1 = 2$ (still reads snapshot `val = 1`).
3. `setVal(prev => prev + 2)` takes the latest batched value ($2$) and adds $2 \rightarrow 4$.
Console logs: **`Render: 4`** (only 1 re-render occurs).

---

### Puzzle 2: The `key` Prop Remount Trap
**What happens to the input text when switching between Patient and Doctor?**
```jsx
function Switcher() {
  const [isDoctor, setIsDoctor] = useState(false);

  return (
    <div>
      <button onClick={() => setIsDoctor(!isDoctor)}>Switch</button>
      {isDoctor ? (
        <input placeholder="Enter Doctor Registration Number" />
      ) : (
        <input placeholder="Enter Patient Phone Number" />
      )}
    </div>
  );
}
```
**Interviewer Question**: If the user types `"9876543210"` into the patient input, then clicks "Switch", does the text disappear or stay?
**Answer**:
**The text STAYS!**
Because both elements are `<input>` at the exact same DOM position in the tree, React preserves the DOM node and only changes the `placeholder` attribute!
**The Senior Fix**: Add unique `key` props (`<input key="doc" ... />` and `<input key="pat" ... />`) to instruct React to unmount the old node and mount a fresh one.

---

## 3. 🏗️ Frontend System Design: "Design an Appointment Booking Slot Lock"

### Interview Question:
> *"When 100 patients try to book the 10:00 AM slot for a popular doctor at the exact same second, how do you handle concurrency, UI responsiveness, and real-time locking?"*

### Architecture Checklist for Full-Stack Engineers:
1. **Optimistic UI Updates**:
   - Immediately mark the slot as `"Selecting..."` on the user's screen.
2. **WebSocket / Server-Sent Events (SSE)**:
   - Broadcast `SLOT_HELD` event to all other clients viewing that doctor's page to disable the slot with a 5-minute countdown.
3. **Backend Redis Distributed Lock**:
   - Backend acquires an atomic Redis lock (`SET lock:doctor:123:slot:1000 my_token NX EX 300`).
4. **Idempotency Keys**:
   - The booking POST request carries a unique UUID so accidental double-taps do not create duplicate appointments.

---

## 4. 🚀 Modern React 19 Essentials: What's New?

### 1. The React Compiler
- In React 19, manual memoization with `useMemo`, `useCallback`, and `React.memo` is increasingly automated. The compiler analyzes JavaScript code and automatically memoizes values that don't need recalculation.

### 2. Actions & `useActionState`
- Streamlines handling async form submissions, replacing manual `isSubmitting`, `error`, and `data` states.

### 3. `useOptimistic`
- Allows developers to render an optimistic UI state before the server finishes responding, rolling back gracefully if the API fails.
