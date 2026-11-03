# `useState` Deep Dive: An Easy-to-Follow Explanation

This guide expands **Section 2, `useState` Deep-Dive**, from the parent lesson. It explains what React remembers, how updates are scheduled, when to use updater functions, and how lazy initialization avoids repeated work.

## Start with the basic idea

`useState` gives a component a value that React remembers between renders, plus a setter that asks React to update that value and render the component again.

```jsx
import { useState } from 'react';

function Counter() {
  const [count, setCount] = useState(0);

  return (
    <button onClick={() => setCount(count + 1)}>
      Clicked {count} times
    </button>
  );
}
```

Here, `0` is the starting value. `count` is the value for this render. `setCount` is the way to request a new value. When the user clicks, React stores the next count and calls `Counter` again; the returned button then shows that new count.

Think of the component function as a fresh calculation each time React renders. React keeps the state between those calculations. A normal local variable does not have that property: it is created again whenever the function runs.

## 1. Read the `useState` declaration

The correct form is:

```js
const [value, setValue] = useState(initialValue);
```

The `useState(...)` call returns an array with two entries. Array destructuring names those entries:

| Name | Meaning |
| --- | --- |
| `value` | The current state snapshot for this render |
| `setValue` | A function that queues a state update |
| `initialValue` | The value used when this component instance first initializes |

For example, `const [isOpen, setIsOpen] = useState(false)` starts closed. Calling `setIsOpen(true)` requests that it become open. State names should describe the information being stored; setter names conventionally start with `set`.

The declaration in the parent lesson, `const [state, setState] = setStateValue`, is only a placeholder-like illustration and is not executable React code. Use `useState(initialValue)` on the right side.

## 2. State updates are scheduled; they do not rewrite this render

Calling a setter does not change the variable in the already-running function. Each render has its own snapshot of props and state. A setter asks React to use a new value on a later render.

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  function handleClick() {
    setCount(count + 1);
    console.log(count); // This handler still sees this render's count.
  }

  return <button onClick={handleClick}>{count}</button>;
}
```

On the first click, the handler reads `count` as `0`, queues `1`, and still logs `0`. React later renders the component with `count` equal to `1`. This is not a delayed mutation of the local variable; it is a new render with a new snapshot.

If you need to do something with the next value inside the handler, calculate it yourself:

```js
const nextCount = count + 1;
setCount(nextCount);
console.log(nextCount); // 1
```

### Batching several updates

React can group related state updates and perform one render after the event handler finishes. This is called batching. It avoids rendering the UI after each individual setter call.

```js
function handleLoad() {
  setLoading(false);
  setDoctors(nextDoctors);
  setError('');
}
```

React can apply all three updates together so the screen moves directly to the consistent loaded state. Batching does not mean setters mutate values immediately: the current handler still sees its original snapshot. In React 18 and later, automatic batching also applies to many updates from promises and timers when using the modern React root API.

## 3. Choose a direct value or a functional updater

Use a direct value when the next state is known independently of the current state:

```js
setSearch('cardiology');
setLoading(true);
```

Use an updater function when the next state depends on the previous state:

```js
setCount(previousCount => previousCount + 1);
```

React supplies the updater with the pending previous value. That makes it reliable when multiple updates are queued before React renders again.

### Why repeated direct increments can surprise you

```js
function addThree() {
  setCount(count + 1);
  setCount(count + 1);
  setCount(count + 1);
}
```

All three expressions use the same `count` snapshot. If it is `0`, each queues `1`, so the result is `1`, not `3`. To apply three increments in sequence, use updater functions:

```js
function addThree() {
  setCount(previous => previous + 1);
  setCount(previous => previous + 1);
  setCount(previous => previous + 1);
}
```

React processes the queued functions in order: `0` becomes `1`, then `2`, then `3`.

For a doctor list, filtering from the previous value is also safer when the update could happen after other state changes:

```js
setDoctors(previousDoctors =>
  previousDoctors.filter(doctor => doctor.id !== removedId)
);
```

An updater should be pure: calculate and return the next state, without changing the previous array or causing side effects. React may call updater functions more than once in development to help reveal impure logic, so do not put API calls, logging-dependent behavior, or other side effects inside them.

## 4. Lazy initial state: defer expensive setup

Usually React ignores the initial value after the first render, but JavaScript still evaluates the argument on every function call:

```js
const [user, setUser] = useState(readUserFromStorage());
```

`readUserFromStorage()` runs every time the component renders, even though React only uses its result for initialization. If the work is expensive, pass a function instead:

```js
const [user, setUser] = useState(() => readUserFromStorage());
```

React calls this initializer to obtain the initial state. The `()` is important: `useState(readUserFromStorage())` calls the function now; `useState(() => readUserFromStorage())` gives React a function to call for initialization.

For storage that may contain invalid JSON, handle parsing errors:

```js
const [user, setUser] = useState(() => {
  try {
    const saved = localStorage.getItem('user');
    return saved ? JSON.parse(saved) : null;
  } catch {
    return null;
  }
});
```

This is useful for initial data such as a saved preference or session hint. It does not automatically keep state synchronized with storage afterward; storage synchronization is a separate requirement.

## A useful mental model

1. React calls the component with its current state snapshot.
2. The component returns the UI for that snapshot.
3. An event handler calls a setter to queue a new state value.
4. React processes queued updates, then calls the component again with the new snapshot.
5. The browser displays the UI produced by that render.

For state derived from an earlier value, use the functional updater. For a fixed next value, pass the value directly. For expensive initial calculation, pass an initializer function.

## Common mistakes and fixes

| Mistake | Why it causes trouble | Better approach |
| --- | --- | --- |
| Expecting the state variable to change immediately after its setter | The handler keeps the render snapshot it started with | Use the next value directly if the handler needs it, or read updated state on a later render |
| Calling a setter during render | It can trigger repeated renders | Call setters in event handlers or appropriate effects |
| Using `setCount(count + 1)` for queued increments | Each update may use the same snapshot | Use `setCount(previous => previous + 1)` |
| Passing an expensive function call as the initial value | JavaScript evaluates it on each render | Pass a function: `useState(() => calculateInitialValue())` |
| Mutating an array/object and passing the same reference back | React may not see a changed reference | Create a new array/object, such as `[...previous, item]` |

## Quick check

If `count` is `4` at the start of one click handler, what does this produce?

```js
setCount(count + 1);
setCount(count + 1);
```

It produces `5`, because both expressions use the current snapshot (`4`). To produce `6`, write:

```js
setCount(previous => previous + 1);
setCount(previous => previous + 1);
```

That difference—**a render snapshot versus a queued update based on the previous value**—is the key idea behind reliable `useState` code.
