# React `children`, Composition, and Slot Props

> **Lesson 3 · React component design**  
> Examples from a Doctor Appointment & Queue Management dashboard

## What you will learn

By the end of this lesson, you should be able to:

- Explain what React puts in the `children` prop.
- Build a reusable wrapper such as `Card`.
- [Describe composition and why it is a natural way to reuse UI in React.](#5-composition-build-by-combining)
- Use named props as slots for specific regions such as an icon or footer.

---

## 1. The dashboard problem

Imagine a doctor dashboard with clinic hours, doctor details, and queue status:

### Today's Clinic Timings

**9:00 AM – 1:00 PM**  
Monday to Saturday

### Doctor Information

**Dr. Amit Sharma**  
General Physician

### Patient Queue

**12 patients**  
Currently waiting

Each section has a similar visual frame: rounded corners, a border, padding, and a background. The information inside each section differs. Repeating the same frame markup and styling three or four times makes later changes harder: if the border or spacing changes, every copy must be updated.

A reusable `Card` can own the shared frame while its caller supplies the content.

## 2. A reusable `Card`

```jsx
export function Card({ children }) {
  return (
    <div className="rounded-card border border-line bg-card p-5">
      {children}
    </div>
  );
}
```

- `Card` is a React component. `export` lets other files import it.
- `({ children })` destructures a prop named `children` from the props object.
- The outer `<div>` owns the shared presentation.
- `{children}` tells React where to render the content supplied by the caller.

The wrapper does not need to know whether it contains a heading, a list, a button, or another component. It is responsible for the frame; the caller is responsible for the content.

## 3. What is the `children` prop?

When JSX is written between a component's opening and closing tags, React makes that content available to the component as `children`.

```jsx
<Card>
  <h2>Today's Clinic Timings</h2>
  <p>9:00 AM – 1:00 PM</p>
</Card>
```

Conceptually, this is similar to passing a props object like this:

```jsx
{
  children: (
    <>
      <h2>Today's Clinic Timings</h2>
      <p>9:00 AM – 1:00 PM</p>
    </>
  )
}
```

This is a mental model, not code you need to write. React supplies the prop. `children` can be JSX elements, text, numbers, or other renderable React content. It can also be empty or contain multiple siblings.

### The box model

```text
Caller provides JSX                Card component
┌────────────────────────┐         ┌───────────────────────────┐
│ <h2>Today's Timings</h2>│  ───▶  │ shared frame              │
│ <p>9:00–1:00</p>       │         │   {children}               │
└────────────────────────┘         └───────────────────────────┘
                                             │
                                             ▼
                                  frame + caller's JSX on screen
```

The key idea is: **content between component tags becomes `children`, and the component chooses where to render it.**

## 4. One `Card`, different content

```jsx
function Dashboard() {
  return (
    <>
      <Card>
        <h2>Today's Clinic Timings</h2>
        <p>9:00 AM – 1:00 PM</p>
        <p>Monday to Saturday</p>
      </Card>

      <Card>
        <h2>Doctor Information</h2>
        <p>Dr. Amit Sharma</p>
        <p>General Physician</p>
      </Card>

      <Card>
        <h2>Patient Queue</h2>
        <p>12 patients currently waiting</p>
      </Card>
    </>
  );
}
```

The same component provides the same frame in all three places. Each use provides different children:

```text
             ┌──────────────┐
             │    Card      │  shared frame
             └──────┬───────┘
        ┌───────────┼───────────┐
        ▼           ▼           ▼
     Timings      Doctor      Queue
      content      content     content
```

## 5. Composition: build by combining

**Composition** means building a larger piece of UI by combining smaller components. Each component can focus on one job, and a parent arranges the pieces.

```jsx
function DoctorDashboard() {
  return (
    <main>
      <Card>
        <DoctorInfo />
      </Card>

      <Card>
        <ClinicTimings />
      </Card>

      <Card>
        <QueueStatus />
      </Card>
    </main>
  );
}
```

Here, `DoctorDashboard` arranges the page, `Card` supplies a shared frame, and the specialized components supply domain-specific content. Think of assembling blocks: each block has a clear purpose, and the arrangement makes a larger interface.

```text
DoctorDashboard
├── Card ── DoctorInfo
├── Card ── ClinicTimings
└── Card ── QueueStatus
```

A useful benefit is that a change to the shared card frame can be made once and used consistently wherever `Card` appears.

## 6. Composition vs. inheritance

In object-oriented programming, **inheritance** lets one class extend another class and receive its behavior or properties:

```js
class Parent {
  greet() {
    console.log("Hello");
  }
}

class Child extends Parent {}

new Child().greet(); // Hello
```

React UI reuse generally does not require a component class to extend another component class. Instead, a component can use another component and supply its content:

```jsx
function Frame({ children }) {
  return <div className="frame">{children}</div>;
}

function Greeting() {
  return (
    <Frame>
      <h1>Hello</h1>
    </Frame>
  );
}
```

`Greeting` composes the `Frame` and provides what goes inside it. It does not inherit the frame's implementation.

| Inheritance | Composition |
|---|---|
| A class extends another class | Components are combined in a render tree |
| Example: `class Child extends Parent` | Example: `<Frame><Greeting /></Frame>` |
| Shares behavior through a class relationship | Reuses UI by wrapping or arranging components and content |
| Common in object-oriented programming | A common way to structure reusable React UI |

**Important:** JavaScript and React can use class components, and JavaScript supports inheritance. The point is that UI reuse in React is usually clearer through composition than through a `Child extends Parent` hierarchy.

## 7. Slot props: named places for content

The `children` prop gives a component one flexible place for caller-provided content. But many UI pieces have several regions with different jobs. An information box, for example, may need an icon beside its title, a main body, and an action area at the bottom.

A **slot** is a place in a component's layout where supplied content will appear. React does not have a special `slot` keyword for this pattern. We create slots using ordinary props, such as `icon`, `title`, and `footer`, and render those props in the right places.

### Start with one content area: `children`

```jsx
function Card({ children }) {
  return <section className="card">{children}</section>;
}
```

Usage:

```jsx
<Card>
  <p>12 patients are waiting.</p>
</Card>
```

Here, the card has one main opening for content. The caller supplies the paragraph, and the component places it inside its frame.

### Add named areas for a more structured component

```jsx
function InfoBox({ title, icon, children, footer }) {
  return (
    <section className="info-box">
      <header className="info-box__header">
        {icon}
        <h2>{title}</h2>
      </header>

      <div className="info-box__body">
        {children}
      </div>

      {footer && (
        <footer className="info-box__footer">
          {footer}
        </footer>
      )}
    </section>
  );
}
```

The component defines four content areas:

| Prop | What it is for | Example value |
|---|---|---|
| `title` | A short heading | `"Today's Clinic Timings"` |
| `icon` | Content next to the heading | `<ClockIcon />` |
| `children` | The main body | `<p>9:00 AM – 1:00 PM</p>` |
| `footer` | Optional supporting action or note | `<button>View Schedule</button>` |

The JSX passed to `icon` and `footer` is sometimes called a **React node**. It can be a component element, HTML element, text, or a combination of renderable content. `title` is a string in this example because the component treats it as text. If you want a richer title area, you could define a separate `titleContent` prop that accepts JSX.

### Use the slots in the clinic dashboard

```jsx
function ClinicTimingsBox() {
  return (
    <InfoBox
      title="Today's Clinic Timings"
      icon={<ClockIcon aria-hidden="true" />}
      footer={<button type="button">View full schedule</button>}
    >
      <p>9:00 AM – 1:00 PM</p>
      <p>Monday to Saturday</p>
    </InfoBox>
  );
}
```

Read this from the caller's point of view:

1. `title` supplies the heading text.
2. `icon` supplies the element that belongs in the header.
3. The content between `<InfoBox>` and `</InfoBox>` becomes `children` and fills the body.
4. `footer` supplies the action at the bottom.

The component controls the arrangement, while the caller chooses the content for each area:

```text
ClinicTimingsBox supplies             InfoBox arranges
──────────────────────────             ────────────────
icon={<ClockIcon />}             ───▶  header icon area
 title="Today's Clinic Timings" ───▶  header title area
 <p>9 AM – 1 PM</p>              ───▶  main body ({children})
 footer={<button>...</button>}   ───▶  footer area
```

This is useful when each part belongs in a known place. The caller can change the icon, message, or action without taking over the whole layout.

### Optional slots and fallback content

A slot can be optional. The earlier example renders the footer only when the caller provides one:

```jsx
{footer && <footer className="info-box__footer">{footer}</footer>}
```

For an icon, the component could instead provide a default:

```jsx
function InfoBox({ title, icon = <InfoIcon aria-hidden="true" />, children, footer }) {
  return (
    <section className="info-box">
      <header className="info-box__header">
        {icon}
        <h2>{title}</h2>
      </header>
      <div className="info-box__body">{children}</div>
      {footer && <footer className="info-box__footer">{footer}</footer>}
    </section>
  );
}
```

Now callers can omit `icon` and the default appears. Use defaults only when that default makes sense for every use of the component.

### When should I use `children` versus a named slot?

Use `children` when the component has one main area and the caller should decide what goes inside it. Use named props when the component has distinct regions with different purposes and the component should place each one deliberately.

```text
Card:       shared frame + one flexible body (children)
InfoBox:    header icon + title + body (children) + optional footer
```

You can use both patterns together, as `InfoBox` does. This keeps the component flexible without making its layout unclear.

### A few practical guidelines

- Choose prop names that describe the region or purpose: `icon`, `footer`, `actions`, `description`.
- Keep simple values such as labels as strings when the component only needs text.
- Pass JSX when callers need to customize markup or provide another component.
- Make optional regions truly optional in the rendered output, or provide a sensible fallback.
- Avoid adding a slot for every tiny detail. If a component has too many unrelated slots, split it into smaller components or use a simpler API.
- Slots receive content; they do not automatically define behavior. For example, a button passed as `footer` still needs an `onClick` handler if it should perform an action.

## 8. Applying the pattern to the project

A doctor appointment dashboard could be organized like this:

```text
DoctorDashboard
├── Card
│   └── DoctorInfo
├── Card
│   └── ClinicTimings
├── Card
│   └── QueueStatus
└── Card
    └── AppointmentList
```

Create the common frame once, then place the relevant content in each card. If the project changes the card's spacing or border, updating the shared component keeps the dashboard consistent. Keep components focused: `Card` handles appearance; `QueueStatus` handles queue-specific presentation and data.

A version with an optional class name might look like this:

```jsx
export function Card({ className = "", children }) {
  return (
    <div className={`rounded-card border border-line bg-card p-5 ${className}`}>
      {children}
    </div>
  );
}
```

This follows the same pattern as a shared `Card` wrapper in a project's UI components. If class names can come from untrusted input or need conflict resolution, use the project's established class-name utility instead of concatenating strings.

## 9. Common misunderstandings

- **“`children` means a single child.”** It can contain multiple sibling elements.
- **“`children` is written explicitly as an attribute.”** Usually it is supplied by the JSX between the component tags.
- **“Slot props are special React syntax.”** They are regular props with names chosen by the component author.
- **“Composition means only using `children`.”** Composition also includes rendering components through named props and arranging components together.
- **“Composition means inheritance is unavailable.”** Inheritance exists in JavaScript; composition is simply the usual fit for reusable React UI.

## 10. Knowledge check

Try answering before reading the answer key.

1. In `<Card><p>Hello</p></Card>`, what does `children` contain?
2. What does `{children}` inside `Card` do?
3. In the `InfoBox` example, why are `icon` and `footer` separate props?
4. How does composition differ from inheritance in the dashboard examples?
5. If you want to change the shared border and padding for every dashboard card, where should that change go?

<details>
<summary><strong>Answer key</strong></summary>

1. The JSX content between `<Card>` and `</Card>`—here, the `<p>Hello</p>` element.
2. It renders the caller-provided content at that location inside the card's output.
3. They represent distinct named regions. The component can place the icon in its header and footer content at the bottom, while `children` remains the main body.
4. Composition combines components and supplied content in the render tree. Inheritance shares class behavior through an `extends` relationship.
5. In the shared `Card` component, so all uses receive the consistent update.

</details>

## Quick recap

- **`children`** is the content passed between a component's opening and closing tags.
- **Composition** builds UI by combining components and supplied content.
- **Slot props** are named props for specific content regions, such as `icon` or `footer`.
- A reusable `Card` owns the shared frame; its caller decides what goes inside.

> **Explain it in your own words:** “A `Card` controls the container, and the caller supplies its children. React composition lets me put the pieces together without making one component inherit from another.”
