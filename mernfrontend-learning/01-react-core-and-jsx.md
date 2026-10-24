# Module 01: React Core Architecture, Virtual DOM & JSX Deep-Dive

---

## 1. 💡 Mental Model: How React Really Works Under The Hood

### Why Not Just Vanilla JavaScript?
In plain JavaScript, updating the UI requires imperative DOM manipulation:
```javascript
// Vanilla JS: Imperative (You describe step-by-step HOW to update the DOM)
const badge = document.getElementById('doctor-badge');
badge.innerText = 'Verified';
badge.classList.add('bg-green-500');
```
As applications grow with hundreds of interconnected components (doctor cards, live tokens, active queues, auth modals), manual DOM manipulation leads to:
1. **State Synchronization Bugs**: Multiple parts of the DOM fall out of sync with application data.
2. **Poor Performance**: Direct DOM reads and writes cause **Layout Thrashing** (forced synchronous reflows & repaints).

### The React Declarative Paradigm
In React, you describe **WHAT** the UI should look like based on the current state. React takes care of the DOM updates:
```jsx
// React: Declarative (UI = f(State))
<Badge verified={doctor.isVerified} />
```

---

## 2. 🔬 Virtual DOM, Fiber Architecture & Reconciliation

### The Real DOM vs The Virtual DOM
* **Real DOM**: Tree of C++ objects managed by the browser engine (Blink, Gecko). Heavyweight; modifying it triggers layout calculations, style recalculations, and repainting.
* **Virtual DOM (VDOM)**: A lightweight in-memory JavaScript representation of the real DOM nodes.

```javascript
// Virtual DOM representation of <h1 className="title">Doctor Profile</h1>
{
  type: 'h1',
  props: {
    className: 'title',
    children: 'Doctor Profile'
  }
}
```

### The 2 Phases of React Rendering:
1. **Render Phase** (Pure & Can be paused/restarted):
   - React calls your component functions.
   - Computes the new Virtual DOM tree.
   - Compares the new tree with the previous tree (**Diffing Algorithm / Reconciliation**).
   - Flags changes (e.g., `Placement`, `Update`, `Deletion`).
2. **Commit Phase** (Synchronous & Cannot be interrupted):
   - React applies the flagged changes to the actual browser DOM using minimal native calls (`appendChild`, `removeChild`, etc.).
   - Runs layout effects (`useLayoutEffect`) and DOM mutation lifecycles.
   - Browser repaints the screen.

### React 16+ Fiber Architecture
Before Fiber (React 15 and earlier), reconciliation was a recursive stack-based process that could block the browser's main thread on large component trees, causing frame drops (jank).
**Fiber** turned the tree traversal into a singly-linked list of work units. It enables **Concurrent Features**—React can pause low-priority work (like rendering an offscreen doctor list) to handle urgent user input (like typing into a search input).

---

## 3. 🔍 JSX Deep-Dive: What is JSX Really?

JSX is **not HTML**. Browsers cannot execute JSX natively. JSX is syntactic sugar that gets compiled by tools like Vite (using `@vitejs/plugin-react` via Babel or esbuild/Rollup).

### Modern JSX Transform (React 17+)
Earlier versions required `import React from 'react'` in every file because JSX compiled to `React.createElement()`.
With the modern transform, Vite automatically imports from `react/jsx-runtime`:

```jsx
// Your Code:
function DoctorBadge({ name }) {
  return <span className="badge">{name}</span>;
}

// What the compiler produces:
import { jsx as _jsx } from 'react/jsx-runtime';

function DoctorBadge({ name }) {
  return _jsx('span', {
    className: 'badge',
    children: name
  });
}
```

### Critical JSX Rules:
1. **Single Root Element**: Every component must return a single top-level node (or a Fragment `<>...</>`). Why? JavaScript functions cannot return two separate objects at once without wrapping them in an array or object.
2. **Attributes are camelCase**: `class` becomes `className` (because `class` is a reserved word in JS), `onclick` becomes `onClick`, `for` becomes `htmlFor`.
3. **Closing Tags are mandatory**: `<img />`, `<input />`, `<br />` must be explicitly closed.

---

## 4. 🏥 Real Project Code Walkthrough: `frontend/src/main.jsx`

Let's look at the actual entry point of our Doctor Appointment App:

```jsx
// frontend/src/main.jsx
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import { BrowserRouter } from 'react-router-dom';
import App from './App.jsx';
import { AuthProvider } from './context/AuthContext.jsx';
import './index.css';

// 1. Target the root div from index.html
const rootElement = document.getElementById('root');

// 2. createRoot initializes the React 18/19 Concurrent Root
createRoot(rootElement).render(
  <StrictMode>
    <BrowserRouter>
      <AuthProvider>
        <App />
      </AuthProvider>
    </BrowserRouter>
  </StrictMode>
);
```

### Architectural Breakdown:
- `createRoot(...)`: Replaces legacy `ReactDOM.render()`. Unlocks React's concurrent rendering capabilities.
- `<StrictMode>`: A developer-only wrapper that helps surface side effects, legacy API usage, and uncleaned timers/listeners.
- Provider Hierarchy:
  - `<BrowserRouter>` provides routing context.
  - `<AuthProvider>` provides global user authentication state.
  - `<App />` handles page views and routes.

---

## 5. ⚠️ Real-World Production War Story

### The Case of the "Disappearing DOM Element"
**Scenario**: In a clinical dashboard, a junior developer wanted to integrate an external chart library (like Chart.js or jQuery datepicker). They wrote:
```javascript
useEffect(() => {
  document.getElementById('chart-container').innerHTML = ''; // DIRECT DOM MUTATION
  renderChart('#chart-container');
}, [data]);
```
**What Went Wrong in Production**:
React maintains its own internal reference to DOM nodes via Fiber nodes. When React attempted to re-render the parent component, it expected the children nodes to still exist. Because vanilla JavaScript wiped out the inner DOM elements behind React's back, React threw:
`NotFoundError: Failed to execute 'removeChild' on 'Node': The node to be removed is not a child of this node.`

**Lesson**: **Never mutate the DOM directly** when React manages the parent container. Always use React refs (`useRef`) to attach third-party non-React libraries to a dedicated unmanaged container.

---

## 6. 🎯 Interview Question Bank: React Core & JSX

### Q1: Why does `useEffect` run twice on component mount in development?
> **Answer**:
> Because `<StrictMode>` intentionally mounts, unmounts, and re-mounts components in development mode.
> React does this to help developers identify missing cleanup logic (such as uncancelled network requests, forgotten `setInterval` timers, or orphaned event listeners). In production builds, this double execution does not happen.

### Q2: What is the difference between `React.createElement` and React Virtual DOM objects?
> **Answer**:
> `React.createElement()` is the factory function that compiles JSX into an immutable JavaScript object (a React element).
> The Virtual DOM is the complete in-memory tree of these React element objects. During reconciliation, React diffs the previous tree against the newly returned tree to determine minimal DOM mutations.

### Q3: Why can't you return multiple sibling elements without a Fragment (`<>...</>`)?
> **Answer**:
> Under the hood, JSX compiles to a function call (`_jsx(...)` or `React.createElement(...)`). In JavaScript, a function call can only return a single value or reference. A Fragment `<>...</>` represents a lightweight container that groups siblings without adding an extra unnecessary `<div>` or wrapper node to the actual browser DOM.

### Q4: What is the diffing algorithm complexity in React, and how did React make it $O(n)$?
> **Answer**:
> The general tree comparison algorithm has a complexity of $O(n^3)$ (comparing 1000 nodes would take a billion operations). React achieved an $O(n)$ heuristic algorithm based on two key assumptions:
> 1. Two elements of different types will produce completely different trees (e.g., changing from `<div>` to `<section>` unmounts the old tree and mounts a fresh one).
> 2. The developer can hint at which child elements are stable across renders using the `key` prop.

### Q5: Output & Reasoning Question
**What is the console output when this code mounts?**
```jsx
function Counter() {
  console.log('A');

  useEffect(() => {
    console.log('B');
    return () => console.log('C');
  }, []);

  console.log('D');
  return <h1>Test</h1>;
}
```
**Answer**:
In Development (with `<StrictMode>`):
1. First render: `A`, `D`
2. Effect mounts: `B`
3. StrictMode cleanup test: `C`
4. Second render: `A`, `D`
5. Effect re-mounts: `B`
Console output: **A -> D -> B -> C -> A -> D -> B**
In Production:
**A -> D -> B**

---

## 7. 💻 Hands-On Mastery Exercise

Inspect `frontend/src/main.jsx`. Try temporarily removing `<StrictMode>` and putting a `console.log("App Mounted")` inside an effect in `frontend/src/App.jsx`. Observe how the logs behave in development!
