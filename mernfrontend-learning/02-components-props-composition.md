# Module 02: Components, Props & Component Composition Patterns

---

## 1. 💡 Mental Model: The Component-Driven Architecture

In modern web development, UI is not a single giant HTML document; it is a tree of isolated, reusable, self-contained **Components**.

```
App
 ├── Navbar
 └── Home
      ├── SearchBar
      ├── FilterSection
      └── DoctorList
           ├── DoctorCard (Dr. Sharma)
           ├── DoctorCard (Dr. Verma)
           └── DoctorCard (Dr. Patel)
```

### Pure Functions & Immutability
A React component is fundamentally a JavaScript function that takes an input object called `props` (properties) and returns JSX:
$$\text{UI} = f(\text{props}, \text{state})$$

> [!IMPORTANT]
> **Props are strictly Read-Only (Immutable)**.
> A component must never modify its own props. If you mutate a prop (`props.doctor.name = 'New Name'`), React will not detect the change reliably, breaking the unidirectional data flow.

---

## 2. 🧩 [Props Deep-Dive](./02-props-deep-dive.md)

This section explains how props are passed between components and how a child receives them.

### 1. Destructuring with Default Values
Rather than accessing `props.width` or `props.className`, modern idiomatic React uses parameter destructuring:

```jsx
// Real example from frontend/src/components/ui.jsx
export function Container({ width = 'max-w-[1240px]', className = '', children }) {
  return (
    <div className={`mx-auto w-full px-4 md:px-6 lg:px-8 ${width} ${className}`}>
      {children}
    </div>
  );
}
```

### 2. Passing Callbacks (Child-to-Parent Communication)
React enforces **Unidirectional Data Flow** (data flows down from parent to child). To send data back up:
1. The **Parent** passes a function prop to the child.
2. The **Child** invokes that function with data as arguments.

```jsx
// Parent:
function DoctorQueue() {
  function handleCallNext(patientId) {
    console.log("Serving patient:", patientId);
  }

  return <PatientRow onCallNext={handleCallNext} patient={currentPatient} />;
}

// Child:
function PatientRow({ patient, onCallNext }) {
  return (
    <button onClick={() => onCallNext(patient.id)}>
      Call Patient
    </button>
  );
}
```

---

## 3. 🏗️ The `children` Prop & Composition vs Inheritance

One of React’s strongest design principles is: **Composition over Inheritance**.
You almost never need `class Child extends Parent`. Instead, you use the `children` prop and specialized slot props.

### Real Project Pattern: The `Card` and `InfoBox` Wrappers
Look at `frontend/src/components/ui.jsx`:
```jsx
export function Card({ className = '', children }) {
  return (
    <div className={`rounded-card border border-line bg-card p-5 ${className}`}>
      {children}
    </div>
  );
}
```

> Helper note: For a deeper walkthrough of `children`, composition, and slot props, see [React `children`, Composition, and Slot Props](./helpher%2001/01-children-composition.md).

Any component can wrap arbitrary content inside `<Card>` without `<Card>` needing to know what is inside:
```jsx
<Card className="shadow-sm">
  <h3>Today's Clinic Timings</h3>
  <p>9:00 AM - 1:00 PM</p>
</Card>
```

---

## 4. 🏥 Real Project Code Walkthrough: `Avatar` & Fallback Handling

Look at how our Doctor App handles doctor profile pictures in `frontend/src/components/ui.jsx`:

```jsx
// frontend/src/components/ui.jsx
function initials(name = '') {
  const words = name.replace(/^dr\.?\s+/i, '').trim().split(/\s+/).filter(Boolean);
  return ((words[0]?.[0] || 'D') + (words[1]?.[0] || '')).toUpperCase();
}

export function Avatar({ src, name, width = 200, className = '' }) {
  const [failedSrc, setFailedSrc] = useState(null);

  // Fallback state: if no image URL was given, or image network request failed
  if (!src || failedSrc === src) {
    return (
      <div 
        role="img" 
        aria-label={name} 
        className={`flex items-center justify-center bg-brand/15 font-serif font-bold text-brand ${className}`}
      >
        {initials(name)}
      </div>
    );
  }

  return (
    <img
      src={optimizeImage(src, width)}
      alt={name}
      loading="lazy"
      onError={() => setFailedSrc(src)}
      className={`object-cover ${className}`}
    />
  );
}
```

### Why this is production-grade:
1. **Graceful Degradation (`onError`)**: If a doctor’s Cloudinary image URL returns a 404 or fails to load, `setFailedSrc(src)` switches the UI to the doctor's initials badge.
2. **Accessibility**: Uses `role="img"` and `aria-label={name}` for screen readers when rendering the fallback initials container.
3. **Performance**: Takes a `width` prop to let Cloudinary transform and downscale the image dynamically, saving mobile bandwidth.

---

## 5. 🏥 Real Project Code Walkthrough: `DoctorCard.jsx`

Look at how `DoctorCard.jsx` breaks down complex logic into small, focused subcomponents:

```jsx
// frontend/src/components/DoctorCard.jsx
function NameWithBadge({ name, verified }) {
  const words = name.split(' ');
  const last = words.pop();
  return (
    <span className="font-serif text-[16.5px] font-semibold">
      {words.join(' ')}{words.length > 0 && ' '}
      <span className="whitespace-nowrap">
        {last}
        {verified && <i className="fas fa-circle-check ml-1.5 text-[13px] text-brand" title="Verified"></i>}
      </span>
    </span>
  );
}

export default function DoctorCard({ doctor }) {
  const minFee = doctor.charges?.length
    ? Math.min(...doctor.charges.map((c) => c.fee))
    : '-';

  return (
    <div className="relative flex flex-col gap-3 rounded-card border border-line bg-card p-[18px]">
      <div className="flex items-start gap-3">
        <Avatar src={doctor.photoUrl} name={doctor.name} width={128} className="size-14 shrink-0 rounded-[14px]" />
        <div>
          <NameWithBadge name={doctor.name} verified={doctor.verified} />
          <div className="text-brand font-bold uppercase">{doctor.specialization}</div>
        </div>
      </div>
    </div>
  );
}
```

### Key Techniques:
- **Subcomponent decomposition**: `NameWithBadge` solves a specific typographic edge-case (preventing the badge icon from wrapping onto its own orphaned line).
- **Safe data traversal**: `doctor.charges?.length` uses Optional Chaining (`?.`) to avoid crashing if `charges` is `undefined`.

---

## 6. ⚠️ Real-World Production War Story

### The "Prop Drilling Explosion" in Multi-Step Forms
**Scenario**: In a hospital management system, patient registration had 5 nested layers:
`RegistrationWizard -> StepThree -> BillingSection -> InsuranceForm -> InsuranceCardInput`.
The developer passed down `patientData`, `setPatientData`, `currency`, `userRole`, and `theme` through every single intermediate component, even though `StepThree` and `BillingSection` never used those properties.

**What Went Wrong**:
When the shape of `userRole` changed, 5 separate files broke. Renaming a prop required manual refactoring across all 5 levels.
**Solution**:
1. Use **Component Composition**: Pass the child ready-made: `<StepThree><InsuranceCardInput /></StepThree>`.
2. Or use **Context API** / state store for truly global state like `theme` or `userRole`.

---

## 7. 🎯 Interview Question Bank: Components & Props

### Q1: What is the difference between Props and State?
> **Answer**:
> - **Props** are arguments passed *into* a component from its parent (like function parameters). They are read-only and immutable to the receiving component.
> - **State** is internal data managed *within* the component (like local variables). When state changes via its updater function, it triggers a re-render of that component and its descendants.

### Q2: What is "Prop Drilling" and what are 3 ways to avoid it?
> **Answer**:
> Prop drilling is the process of passing props down through multiple layers of intermediate components that do not need the data themselves, merely to reach a deeply nested child.
> 3 ways to eliminate it:
> 1. **Component Composition**: Pass `<NestedChild />` as `children` or as a prop directly from the top level.
> 2. **React Context API**: Provide data at a parent level with `createContext` and consume it with `useContext`.
> 3. **Global State Libraries**: Use Zustand, Redux Toolkit, or Jotai for cross-cutting state.

### Q3: Why can’t we mutate props directly? (e.g., `props.total += 10`)
> **Answer**:
> React relies on pure functions and shallow comparison of objects for reconciliation. If you mutate `props.total`, the reference to the parent's state object might remain the same, causing React to not re-render as expected.

### Q4: Code Output & Concept Check
**What will be displayed on the screen?**
```jsx
function Greeting({ name }) {
  name = name || "Doctor";
  return <h1>Hello, {name}</h1>;
}

export default function App() {
  return <Greeting name="" />;
}
```
> **Answer**:
> **`Hello, Doctor`**
> Because `name` is an empty string `""`, which is falsy in JavaScript, the `||` operator evaluates to `"Doctor"`.
> **Follow-up Interview Question**: If `name` was `null` or `undefined` with default props `function Greeting({ name = "Doctor" })`, what happens if you pass `<Greeting name="" />`?
> With default parameter `{ name = "Doctor" }`, an empty string `""` is **not** `undefined`, so it would display `Hello, ` (empty)!

---

## 8. 💻 Hands-On Mastery Exercise

Look at `frontend/src/components/ui.jsx`. 
Create a new reusable component:
```jsx
export function Badge({ variant = 'success', children }) { ... }
```
Supporting variants: `'success'` (green), `'warning'` (amber), `'danger'` (red). Use it in `DoctorCard.jsx` to show whether a doctor is "Available Today" or "On Leave".
