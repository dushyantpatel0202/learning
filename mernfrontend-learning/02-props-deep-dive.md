# Props Deep-Dive: How Props Move From One File to Another

This file explains the two topics covered under the Props Deep-Dive section:

1. Destructuring with default values
2. Passing callbacks from child to parent

This is the detailed companion to the main lesson in [02-components-props-composition.md](./02-components-props-composition.md).

---

## 1) How props pass from one file to another

In React, a parent component creates data and passes it into a child component using JSX attributes.

### Example: Parent file passes a prop to child file

`DoctorList.jsx`
```jsx
import DoctorCard from './DoctorCard';

export default function DoctorList() {
  const doctor = {
    name: 'Dr. Sharma',
    specialization: 'Cardiologist',
    verified: true,
  };

  return <DoctorCard doctor={doctor} />;
}
```

`DoctorCard.jsx`
```jsx
export default function DoctorCard({ doctor }) {
  return (
    <div>
      <h3>{doctor.name}</h3>
      <p>{doctor.specialization}</p>
    </div>
  );
}
```

### What is happening here?

1. In the parent file (`DoctorList.jsx`), we create an object named `doctor`.
2. We pass it into the child using this syntax:
   ```jsx
   <DoctorCard doctor={doctor} />
   ```
3. React collects that value as a prop object.
4. In the child file (`DoctorCard.jsx`), the function receives it as:
   ```jsx
   function DoctorCard({ doctor })
   ```
5. This is called destructuring. It means:
   - `doctor` is the prop value
   - the child can use it directly without writing `props.doctor`

### How the child receives the prop?

When React calls a component, it internally does something like this:

```jsx
function DoctorCard(props) {
  const { doctor } = props;

  return (
    <div>
      <h3>{doctor.name}</h3>
    </div>
  );
}
```

So the child is not reading a global variable. It is reading the value passed in from the parent.

---

## 2) Destructuring with default values

This is the modern React style. Instead of doing `props.width` and `props.className`, we destructure the props directly in the function parameter.

### Example

```jsx
export function Container({
  width = 'max-w-[1240px]',
  className = '',
  children,
}) {
  return (
    <div className={`mx-auto w-full px-4 ${width} ${className}`}>
      {children}
    </div>
  );
}
```

### Why this is useful

- shorter and cleaner code
- default values protect against missing props
- you can pass a custom `width` or fallback to a safe default

### Example usage

```jsx
<Container>Welcome to the clinic dashboard</Container>
<Container width="max-w-[900px]" className="bg-white">Appointments</Container>
```

If `width` is not passed, this will use:
```jsx
'max-w-[1240px]'
```

If `className` is not passed, this will use:
```jsx
''
```

---

## 3) Passing callbacks from child back to parent

Props are not only used to pass data down. They can also pass functions down so that a child can notify its parent.

This is the pattern used for child-to-parent communication.

### Example: Parent passes a function to child

`AppointmentPage.jsx`
```jsx
import PatientRow from './PatientRow';

export default function AppointmentPage() {
  function handleCallNext(patientId) {
    console.log('Serving patient:', patientId);
  }

  return <PatientRow patient={{ id: 12, name: 'Rahul' }} onCallNext={handleCallNext} />;
}
```

### Child receives the function and calls it

`PatientRow.jsx`
```jsx
export default function PatientRow({ patient, onCallNext }) {
  return (
    <button onClick={() => onCallNext(patient.id)}>
      Call Patient
    </button>
  );
}
```

### Flow of execution

1. Parent creates `handleCallNext`.
2. Parent passes it as a prop:
   ```jsx
   <PatientRow onCallNext={handleCallNext} />
   ```
3. Child receives it as `onCallNext`.
4. Child triggers it when needed:
   ```jsx
   onClick={() => onCallNext(patient.id)}
   ```
5. Parent function runs and handles the data.

This is the core idea of unidirectional data flow:
- data and functions move down from parent to child
- child sends information back by calling the function passed by parent

---

## 4) Visual flow: parent to child

```text
Parent File (DoctorList.jsx)
      |
      |  doctor={doctor}
      v
Child File (DoctorCard.jsx)
      |
      |  receives { doctor }
      v
Uses doctor.name, doctor.specialization
```

This means the parent owns and decides the data, while the child simply reads it.

---

## 5) Important rules about props

### Rule 1: Props are read-only

A child component should never directly mutate props.

```jsx
function DoctorCard({ doctor }) {
  doctor.name = 'New Name'; // ❌ bad
  return <h3>{doctor.name}</h3>;
}
```

This breaks the principle of data flow and can create bugs.

### Rule 2: The child only gets what the parent passes

If the parent does not pass a prop, the child cannot use it unless it has a default value.

```jsx
function Greeting({ name = 'Doctor' }) {
  return <h1>Hello, {name}</h1>;
}
```

### Rule 3: A component can receive multiple props

```jsx
function DoctorCard({ doctor, showBadge, onClick }) {
  return (
    <button onClick={onClick}>
      {doctor.name}
      {showBadge && ' ✅'}
    </button>
  );
}
```

---

## 6) Real-world example in a doctor app

Here is a practical version using a parent and child:

`DoctorList.jsx`
```jsx
import DoctorCard from './DoctorCard';

export default function DoctorList() {
  const doctors = [
    { id: 1, name: 'Dr. Sharma', specialization: 'Cardiologist', verified: true },
    { id: 2, name: 'Dr. Verma', specialization: 'Dermatologist', verified: false },
  ];

  return (
    <div>
      {doctors.map((doctor) => (
        <DoctorCard key={doctor.id} doctor={doctor} />
      ))}
    </div>
  );
}
```

`DoctorCard.jsx`
```jsx
export default function DoctorCard({ doctor }) {
  return (
    <div>
      <h3>{doctor.name}</h3>
      <p>{doctor.specialization}</p>
      {doctor.verified && <span>Verified</span>}
    </div>
  );
}
```

### This is exactly how prop flow works

- `DoctorList` owns the list of doctors
- it sends each `doctor` to `DoctorCard`
- `DoctorCard` receives it as a prop and renders the UI

---

## 7) Quick interview answer

If asked: "How do props pass from one file to another in React?"

Answer:

> In React, a parent component passes values to a child component using JSX attributes. The child receives them as function parameters or as `props`. The child can then destructure the values and render them.

---

## 8) Practice question

Try writing your own component flow:

```jsx
function Parent() {
  const doctor = { name: 'Dr. Patel' };
  return <Child doctor={doctor} />;
}

function Child({ doctor }) {
  return <h2>{doctor.name}</h2>;
}
```

If you understand this pattern, you understand the heart of React component communication.

---

## 9) Summary

- Props are values passed from a parent component to a child component.
- The parent writes the value in JSX.
- The child receives it in its function parameter.
- Destructuring makes it cleaner and easier to read.
- Callback props allow the child to communicate back to the parent.
- Props are read-only and should not be mutated.

This is the foundation of reusable, component-based UI architecture in React.
