# DoctorCard: A Step-by-Step React Walkthrough

This lesson traces how a doctor data object becomes a reusable card in the Doctor Appointment App. It focuses on props, small subcomponents, derived values, optional chaining, conditional rendering, and layout classes.

## 1. The component's responsibility

`DoctorCard` receives one doctor and renders a compact summary. Keeping this UI in a component makes it reusable wherever a list of doctors appears. The parent owns the doctor data and passes it down as a prop; `DoctorCard` reads that data to render the UI.

## 2. A focused subcomponent for the name and badge

```jsx
function NameWithBadge({ name, verified }) {
  const words = name.split(' ');
  const last = words.pop();

  return (
    <span className="font-serif text-[16.5px] font-semibold">
      {words.join(' ')}{words.length > 0 && ' '}
      <span className="whitespace-nowrap">
        {last}
        {verified && (
          <i
            className="fas fa-circle-check ml-1.5 text-[13px] text-brand"
            title="Verified"
          />
        )}
      </span>
    </span>
  );
}
```

The component receives `name` and `verified` as props. It separates the last word from the rest of the name so the verified icon can stay beside that word. The `whitespace-nowrap` class keeps the last name and badge together, avoiding an orphaned icon on a separate line.

`words.pop()` returns and removes the last element of the array. `words.join(' ')` joins the remaining name parts with spaces. The expression `{words.length > 0 && ' '}` adds a separating space only when there are preceding name parts.

The verified icon is conditionally rendered:

```jsx
{verified && <i title="Verified" />}
```

If `verified` is truthy, React renders the icon. If it is false, React renders nothing for that expression. The `title` attribute can expose a tooltip, though production interfaces should also make verification status clear to screen readers where needed.

## 3. Receive the doctor as a prop

```jsx
export default function DoctorCard({ doctor }) {
  // render a card using doctor fields
}
```

The parent can use it like this:

```jsx
<DoctorCard doctor={doctor} />
```

Destructuring `{ doctor }` is JavaScript syntax for reading the `doctor` property from the props object. Props are inputs owned by the parent; the card should not mutate them.

## 4. Calculate a display value from the data

```jsx
const minFee = doctor.charges?.length
  ? Math.min(...doctor.charges.map((charge) => charge.fee))
  : '-';
```

This derives the smallest fee from the doctor's list of charges. `doctor.charges?.length` uses optional chaining: if `charges` is `null` or `undefined`, the expression safely evaluates to `undefined` instead of throwing. If the array has entries, `map` extracts each `fee`; the spread operator passes those numbers as individual arguments to `Math.min`. If the array is missing or empty, the UI value becomes `'-'`.

Derived values should be calculated from current props rather than copied into state when they do not need independent state. This keeps the display synchronized with the latest doctor data.

> Note: The example below computes `minFee`, but the simplified JSX shown here does not render it. A production card would display it or remove the unused calculation.

## 5. Compose the card from smaller elements

```jsx
export default function DoctorCard({ doctor }) {
  const minFee = doctor.charges?.length
    ? Math.min(...doctor.charges.map((charge) => charge.fee))
    : '-';

  return (
    <div className="relative flex flex-col gap-3 rounded-card border border-line bg-card p-[18px]">
      <div className="flex items-start gap-3">
        <Avatar
          src={doctor.photoUrl}
          name={doctor.name}
          width={128}
          className="size-14 shrink-0 rounded-[14px]"
        />
        <div>
          <NameWithBadge name={doctor.name} verified={doctor.verified} />
          <div className="text-brand font-bold uppercase">
            {doctor.specialization}
          </div>
        </div>
      </div>
    </div>
  );
}
```

The outer `div` defines the card surface and vertical spacing. The inner row places the avatar and text side by side. `Avatar` handles its own image fallback behavior; `NameWithBadge` handles the name and verified marker. This keeps `DoctorCard` focused on arranging the pieces.

The card passes only the values each child needs. For example, the avatar gets the photo URL and display name, while `NameWithBadge` gets the name and verification state. This is a simple form of component composition: build a larger interface by combining focused components.

## 6. Key techniques

- **Subcomponent decomposition:** `NameWithBadge` isolates name formatting and badge behavior.
- **Safe data access:** Optional chaining prevents an error if `charges` is absent.
- **Derived data:** The minimum fee is calculated from current props.
- **Conditional rendering:** The verification icon appears only when `verified` is truthy.
- **Composition:** The card combines `Avatar` and `NameWithBadge` rather than duplicating their logic.
- **Layout:** Flexbox and spacing classes align the avatar and text.

## 7. Check your understanding

1. Why does `NameWithBadge` receive `name` and `verified` instead of the whole doctor object? **It only needs those two values, which makes its inputs and responsibility clearer.**
2. What does `doctor.charges?.length` protect against? **It avoids reading `length` from a missing `charges` value.**
3. When does React render the verified icon? **When `verified` is truthy.**
4. Why extract `NameWithBadge`? **It gives the name formatting a focused, reusable component.**

## Key takeaway

`DoctorCard` is easier to maintain when it delegates specialized display behavior to small components, derives simple values from props, and keeps data ownership in the parent.

---

## Deeper walkthrough: follow the code from props to UI

The examples below walk through each JavaScript expression and layout choice in more detail. The names and card code are illustrative examples based on the project pattern.

### 1. Follow `NameWithBadge`

```jsx
function NameWithBadge({ name, verified }) {
  const words = name.split(' ');
  const last = words.pop();

  return (
    <span className="font-serif text-[16.5px] font-semibold">
      {words.join(' ')}{words.length > 0 && ' '}
      <span className="whitespace-nowrap">
        {last}
        {verified && (
          <i
            className="fas fa-circle-check ml-1.5 text-[13px] text-brand"
            title="Verified"
          />
        )}
      </span>
    </span>
  );
}
```

Imagine the inputs are `name="Dr. Rahul Kumar"` and `verified={true}`. The component receives a props object, and `{ name, verified }` is JavaScript object destructuring. It is equivalent to reading `props.name` and `props.verified` inside a function that accepts `props`.

#### `split(' ')`: string to array

```js
const words = 'Dr. Rahul Kumar'.split(' ');
// ['Dr.', 'Rahul', 'Kumar']
```

The array indices are `0` for `Dr.`, `1` for `Rahul`, and `2` for `Kumar`. `split(' ')` divides only at a single ordinary space. If names may contain repeated whitespace, `split(/\s+/)` after trimming is more robust.

#### `pop()`: remove and return the last word

```js
const words = ['Dr.', 'Rahul', 'Kumar'];
const last = words.pop();
// last is 'Kumar'
// words is now ['Dr.', 'Rahul']
```

`pop()` changes the array it is called on and returns the removed item. Here, the last name is separated so it can stay beside the verification icon.

#### `join()`: array back to a string

```js
words.join(' '); // 'Dr. Rahul'
```

The JSX renders that first portion and adds a space only if it exists:

```jsx
{words.join(' ')}{words.length > 0 && ' '}
```

The `&&` operator evaluates its right side only when the left side is truthy. If a one-word name is passed, `pop()` leaves an empty array; no extra leading space is rendered.

#### Keep the badge beside the last word

The last word and icon share a span with Tailwind's `whitespace-nowrap`. This prevents a narrow line from separating the badge from the word it describes. Without it, the final word might appear at the end of one line while the check icon wraps alone to the next.

#### Render the icon only for verified doctors

```jsx
{verified && <i className="fas fa-circle-check ..." title="Verified" />}
```

When `verified` is truthy, React renders the icon element. When it is false, the expression produces no icon. The `fas` and `fa-circle-check` classes come from Font Awesome; `ml-1.5`, `text-[13px]`, and `text-brand` style its spacing, size, and color. The `title` can provide a hover tooltip, but a real product should ensure the verification meaning is also accessible without relying only on a tooltip or icon.

### 2. Follow `DoctorCard` props

```jsx
export default function DoctorCard({ doctor }) {
  // use the doctor object to render one card
}
```

The parent might render `<DoctorCard doctor={doctorData} />`. The component receives the complete object through its `doctor` prop. For example:

```js
const doctorData = {
  name: 'Dr. Rahul Kumar',
  verified: true,
  photoUrl: '/images/rahul.jpg',
  specialization: 'Cardiologist',
  charges: [
    { service: 'Consultation', fee: 500 },
    { service: 'Follow-up', fee: 300 },
    { service: 'Video Consultation', fee: 400 },
  ],
};
```

Props are inputs from the parent. The card should read them, not mutate them. Destructuring the `doctor` property is ordinary JavaScript syntax.

### 3. Calculate the minimum fee

```js
const minFee = doctor.charges?.length
  ? Math.min(...doctor.charges.map((charge) => charge.fee))
  : '-';
```

For the example data, the fee values are `500`, `300`, and `400`, so `minFee` becomes `300`.

#### Optional chaining: `doctor.charges?.length`

Without optional chaining, `doctor.charges.length` throws if `charges` is `undefined` or `null`. With `?.`, JavaScript checks `charges` first. If it is missing, the expression returns `undefined` safely. If it exists, JavaScript reads its `length`.

This check also treats an empty array as the fallback case because its length is `0`, which is falsy. Optional chaining here does **not** validate that `charges` is an array, that each item has a numeric fee, or that `doctor` itself exists. Validate data at the API boundary and use defensive rendering when the component can receive incomplete data.

#### Ternary: choose a value

The pattern `condition ? valueIfTrue : valueIfFalse` is a compact `if/else`. Here, a non-empty charges array triggers the calculation; otherwise `minFee` is `'-'`.

#### `map()`: make an array of fees

```js
doctor.charges.map((charge) => charge.fee);
// [500, 300, 400]
```

`map()` creates a new array by running the callback once for each item. The callback `(charge) => charge.fee` extracts one fee from each charge object. The original `charges` array is not changed.

#### Spread syntax and `Math.min()`

`Math.min()` takes separate numeric arguments, so the spread syntax unpacks the array:

```js
Math.min(...[500, 300, 400]);
// equivalent to Math.min(500, 300, 400)
// 300
```

In short: `map()` transforms the array, `...` spreads its values into arguments, and `Math.min()` finds the smallest number.

> In the short project snippet above, `minFee` is calculated but not rendered in the JSX. If the real component also does not use it, remove the calculation or display it in the card to avoid unused work and lint warnings.

### 4. Read the JSX layout from outside in

```jsx
return (
  <div className="relative flex flex-col gap-3 rounded-card border border-line bg-card p-[18px]">
    <div className="flex items-start gap-3">
      <Avatar
        src={doctor.photoUrl}
        name={doctor.name}
        width={128}
        className="size-14 shrink-0 rounded-[14px]"
      />
      <div>
        <NameWithBadge name={doctor.name} verified={doctor.verified} />
        <div className="text-brand font-bold uppercase">
          {doctor.specialization}
        </div>
      </div>
    </div>
  </div>
);
```

The outer `div` is the card container. `relative` establishes a positioning context for any absolutely positioned child. `flex flex-col` creates a vertical flex layout, `gap-3` spaces its children, `rounded-card` rounds its corners, `border` and `border-line` define the outline, `bg-card` sets the background, and `p-[18px]` adds 18 pixels of padding.

The inner `div` uses `flex` to place the avatar and details side by side, `items-start` to align them at the top, and `gap-3` to create space between them.

`Avatar` receives the doctor's photo URL and name. The earlier Avatar lesson explains its initials fallback. In `className`, `size-14` sets a 56-pixel square, `shrink-0` prevents flexbox from shrinking it, and `rounded-[14px]` gives it rounded corners.

`NameWithBadge` receives only the two values it needs. The specialization is rendered separately; Tailwind's `uppercase` changes its visual presentation, not the original JavaScript string.

### 5. Concepts at a glance

| Concept | What it does here |
| --- | --- |
| Destructuring | Reads props such as `name`, `verified`, and `doctor` |
| `split()` | Turns a name string into an array of words |
| `pop()` | Removes and returns the final word |
| `join()` | Combines the remaining words into a string |
| `&&` | Conditionally renders the verification icon or a space |
| `?.` | Safely reads `length` when `charges` may be missing |
| `map()` | Extracts fee values from charge objects |
| `...` | Passes array values as individual function arguments |
| `Math.min()` | Finds the smallest fee |
| Ternary `? :` | Chooses the fee or a fallback dash |
| Component composition | Builds the card from focused child components |

### 6. Data assumptions and production improvements

This example assumes `doctor` and `doctor.name` exist, and that each charge has a valid numeric `fee`. It also assumes `charges` is an array when present. Validate API data and decide how the UI should behave for incomplete values. For names with repeated spaces, normalize whitespace before splitting. If an icon communicates verification, expose that status accessibly (for example, with meaningful text or an accessible label) rather than relying only on its visual shape.

### 7. Knowledge check

1. What does `split(' ')` return? **An array of pieces from the string.**
2. What does `words.pop()` do? **It removes and returns the last array element.**
3. What does `doctor.charges?.length` guard against? **Accessing `length` when `charges` is `null` or `undefined`.**
4. What does `map((charge) => charge.fee)` return? **A new array containing the fee from each charge.**
5. Why use `...` in `Math.min(...fees)`? **To pass array values as separate arguments to `Math.min()`.**

Try explaining `pop()`, optional chaining, `map()`, and spread syntax without reading the answers. These JavaScript patterns appear throughout React applications.
