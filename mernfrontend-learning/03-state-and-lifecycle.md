# Module 03: State Management & Lifecycle (`useState`, `useEffect`)

---

## 1. 💡 Mental Model: What is State?

In React, **State** is the component's private memory. When a user interacts with a page (types into a search input, clicks "Book Appointment", or switches tabs), the component updates its state.

Updating state triggers a **Re-render**:
1. React executes your component function again with the fresh state value.
2. Creates the new Virtual DOM tree.
3. Diffs it with the old tree and updates only the changed DOM elements.

---

## 2. 🔬 `useState` Deep-Dive

### 1. The Anatomy of `useState`
```javascript
const [state, setState] = setStateValue;
```

### 2. The Asynchronous Nature & Automatic Batching (React 18+)
State setters do not change the variable immediately in place!

```javascript
// ❌ WRONG EXPECTATION:
const [count, setCount] = useState(0);

function handleClick() {
  setCount(count + 1);
  console.log(count); // Still logs 0! Not 1!
}
```
**Why?** A render snapshot freezes variables during that execution. `count` is a `const` for that specific render pass.

#### Automatic Batching:
In React 18/19, multiple state updates inside click handlers, `setTimeout`, promises, or native events are batched into a single re-render to avoid performance lag.

```javascript
// Triggers ONLY 1 re-render:
function handleUpdate() {
  setLoading(false);
  setDoctors(newList);
  setError('');
}
```

### 3. Direct Update vs Functional Updater (`prev => ...`)
When your next state depends on the previous state, **always use the functional form**:

```javascript
// ❌ Buggy: If called rapidly in succession, both read the same snapshot:
setDoctors(doctors.filter(...));

// ✅ Safe & Guaranteed: Always receives the most up-to-date state:
setDoctors(prev => prev.filter(doc => doc.id !== removedId));
```

### 4. Lazy Initial State
If computing the initial state is expensive (e.g., reading from `localStorage`), pass a function to `useState`, not a direct value:

```javascript
// ❌ BAD: JSON.parse runs on EVERY single re-render:
const [user, setUser] = useState(JSON.parse(localStorage.getItem('user')));

// ✅ GOOD (Lazy): Function runs ONLY ONCE during initial mount:
const [user, setUser] = useState(() => {
  try {
    return JSON.parse(localStorage.getItem('user')) || null;
  } catch {
    return null;
  }
});
```

*(See this exact pattern implemented in `frontend/src/context/AuthContext.jsx` lines 17-26!)*

---

## 3. 🔄 Updating Objects & Arrays (The Immutability Rule)

Never mutate an object or array directly in state:
```javascript
// ❌ BAD (Mutates state directly; React won't detect reference change):
doctors.push(newDoctor);
setDoctors(doctors);

// ✅ GOOD (Creates a brand new array reference):
setDoctors(prev => [...prev, newDoctor]);

// ✅ Updating an item inside an array:
setDoctors(prev => prev.map(doc => 
  doc._id === targetId ? { ...doc, isAvailable: false } : doc
));
```

---

## 4. ⚡ `useEffect` Deep-Dive & The Component Lifecycle

A **Side Effect** is any operation that reaches outside the pure React rendering process (e.g., fetching data from an API, subscribing to WebSockets, setting browser titles, reading geolocation).

```javascript
useEffect(() => {
  // 1. Setup code (runs on mount or when dependencies change)

  return () => {
    // 2. Cleanup code (runs on unmount or before the effect runs again)
  };
}, [dependencies]);
```

### The 3 Dependency Variations:
| Dependency Array | When it runs | Analogy |
| :--- | :--- | :--- |
| `useEffect(() => {})` (No array) | Runs after **every single render** | High risk of infinite loops! |
| `useEffect(() => {}, [])` (Empty array) | Runs **once** after initial mount | `componentDidMount` |
| `useEffect(() => {}, [search, page])` | Runs on mount + whenever `search` or `page` changes | `componentDidUpdate` |

---

## 5. 🏥 Real Project Code Walkthrough: `frontend/src/pages/Home.jsx`

Let's inspect how `Home.jsx` handles state, API loading, and side-effects:

```jsx
// frontend/src/pages/Home.jsx
export default function Home() {
  const [doctors, setDoctors] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState('');
  const [search, setSearch] = useState('');
  const [sortedByDistance, setSortedByDistance] = useState(false);

  // 1. Fetch doctors once when the home page first loads:
  useEffect(() => {
    fetchDoctors();
  }, []);

  async function fetchDoctors(params = {}) {
    try {
      setLoading(true);
      const res = await api.get('/doctors', { params });
      setDoctors(res.data);
      setSortedByDistance(false);
      setError('');
    } catch (err) {
      setError('Could not load doctors. Is the backend server running?');
    } finally {
      // Guaranteed to turn off spinner even if API fails!
      setLoading(false);
    }
  }

  // 2. Functional state update when calculating distances:
  function handleNearMe() {
    navigator.geolocation.getCurrentPosition(pos => {
      const { latitude, longitude } = pos.coords;
      setDoctors(prev => {
        const withDistance = prev.map(d => ({
          ...d,
          distanceKm: distanceInKm(latitude, longitude, d.latitude, d.longitude),
        }));
        withDistance.sort((a, b) => a.distanceKm - b.distanceKm);
        return withDistance;
      });
      setSortedByDistance(true);
    });
  }
}
```

### Why this is production-grade:
1. **`finally { setLoading(false); }`**: If the server is down or throws a 500, the user will see an error message instead of an infinite loading spinner.
2. **Functional update in `setDoctors(prev => ...)`**: The geolocation callback is asynchronous and might resolve seconds later. Using `prev` guarantees we sort the latest doctor data rather than an outdated closure variable.

---

## 6. ⚠️ Real-World Production War Story: The Search Race Condition Bug

### The Problem:
Imagine a patient typing `"Dentist"` into the search box:
1. User types `"Den"` $\rightarrow$ Request 1 starts (takes 600ms).
2. User types `"Dentist"` $\rightarrow$ Request 2 starts (faster network route, takes 200ms).
3. Request 2 completes first $\rightarrow$ Doctors matching `"Dentist"` appear.
4. Request 1 completes second $\rightarrow$ **Doctors matching `"Den"` overwrite the UI!** The user typed "Dentist", but sees results for "Den"!

### The Senior Engineer Solution: `AbortController` in `useEffect` Cleanup
```jsx
useEffect(() => {
  const controller = new AbortController();

  async function search() {
    try {
      setLoading(true);
      const res = await api.get('/doctors', {
        params: { search },
        signal: controller.signal // Link axios request to the abort signal
      });
      setDoctors(res.data);
    } catch (err) {
      if (err.name !== 'CanceledError') {
        setError(err.message);
      }
    } finally {
      setLoading(false);
    }
  }

  if (search.trim()) {
    search();
  }

  // CLEANUP FUNCTION: If search changes before response arrives, cancel previous request!
  return () => {
    controller.abort();
  };
}, [search]);
```

---

## 7. 🎯 Interview Question Bank: `useState` & `useEffect`

### Q1: What causes an infinite loop in `useEffect`?
> **Answer**:
> Updating a state variable inside `useEffect` that is also listed in that effect's dependency array (or omitting the dependency array entirely):
> ```javascript
> useEffect(() => {
>   setCount(count + 1); // Triggers re-render -> runs effect again -> triggers re-render...
> }, [count]);
> ```

### Q2: What is the purpose of the cleanup function in `useEffect`? Give 3 real examples.
> **Answer**:
> The cleanup function runs when the component unmounts and before the effect re-runs on a dependency change.
> 1. **Canceling pending API requests** (via `AbortController`) to avoid race conditions.
> 2. **Clearing timers** (`clearInterval`, `clearTimeout`) to prevent memory leaks.
> 3. **Unsubscribing from event listeners / WebSockets** (`window.removeEventListener('resize', ...)`).

### Q3: Output & Closure Puzzle
**What will be logged when the button is clicked 3 times rapidly?**
```jsx
function Counter() {
  const [count, setCount] = useState(0);

  function handleClick() {
    setTimeout(() => {
      console.log(count);
    }, 3000);
    setCount(count + 1);
  }

  return <button onClick={handleClick}>Click</button>;
}
```
> **Answer**:
> If clicked rapidly 3 times (at t = 0s, 0.2s, 0.4s):
> At t = 3s: logs **`0`**
> At t = 3.2s: logs **`1`**
> At t = 3.4s: logs **`2`**
> **Why?** Due to JavaScript closures, each click handler captured the snapshot of `count` from the render in which the click occurred (`0`, then `1`, then `2`).

### Q4: When should you NOT use `useEffect`?
> **Answer**:
> 1. **Transforming data for rendering**: Do not put computed state in `useEffect`. Calculate it directly during render:
>    ```javascript
>    // ❌ BAD:
>    useEffect(() => { setFullName(firstName + ' ' + lastName); }, [firstName, lastName]);
>    // ✅ GOOD:
>    const fullName = `${firstName} ${lastName}`;
>    ```
> 2. **Handling user interactions**: Action logic belongs in event handlers (`onClick`, `onSubmit`), not in `useEffect`.

---

## 8. 💻 Hands-On Mastery Exercise

Look at `fetchDoctors` in `frontend/src/pages/Home.jsx`. Refactor it to handle component unmounting and rapid search typing using `AbortController` as shown in Section 6.
