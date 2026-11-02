# Avatar Fallback Handling: A Step-by-Step React Walkthrough

This lesson follows the `Avatar` component used for doctor profile pictures. If the image loads, the component displays it. If the URL is missing or the image request fails, it displays the doctor's initials instead. This is **fallback handling**: providing an alternative when the primary resource is unavailable.

## 1. What the user sees

| Doctor | Image result | UI |
| --- | --- | --- |
| Dr. Rahul Sharma | Image loads | Profile photo |
| Dr. Amit Patel | Image fails | Initials `AP` |

The fallback avoids a broken-image icon and keeps the avatar useful in the layout.

## 2. Generate initials from a name

```js
function initials(name = '') {
  const words = name
    .replace(/^dr\.?\s+/i, '')
    .trim()
    .split(/\s+/)
    .filter(Boolean);

  return (
    (words[0]?.[0] || 'D') +
    (words[1]?.[0] || '')
  ).toUpperCase();
}
```

The function removes an optional `Dr` prefix, splits the remaining name into words, takes the first character of the first two words, and converts the result to uppercase.

| Input | Result |
| --- | --- |
| `Dr. Rahul Sharma` | `RS` |
| `Dr Amit Patel` | `AP` |
| `Priya Verma` | `PV` |
| `Rahul` | `R` |
| No name | `D` |

`name = ''` is a JavaScript default parameter. It supplies an empty string if no argument is passed. The regular expression `/^dr\.?\s+/i` matches `Dr` or `Dr.` at the beginning of the string, followed by whitespace. `^` means start of string, `\.?` means an optional period, `\s+` means one or more whitespace characters, and `i` makes the match case-insensitive.

`trim()` removes whitespace at both ends. `split(/\s+/)` divides the name into words. `filter(Boolean)` removes empty strings that could result from repeated whitespace:

```js
['Rahul', '', 'Sharma'].filter(Boolean);
// ['Rahul', 'Sharma']
```

JavaScript arrays start at index zero, so `words[0]` is the first word. `words[0]?.[0]` safely reads its first character. Optional chaining returns `undefined` if the word is missing instead of throwing. The `||` operator supplies a fallback character if the value is empty or missing. For `Dr. Rahul Sharma`, the words are `['Rahul', 'Sharma']`, so the result is `RS`.

## 3. The Avatar component and its props

```jsx
export function Avatar({ src, name, width = 200, className = '' }) {
  // component body
}
```

| Prop | Purpose | Default |
| --- | --- | --- |
| `src` | Doctor's image URL | None |
| `name` | Doctor's name and image description | None |
| `width` | Requested image transformation width | `200` |
| `className` | Additional CSS classes | Empty string |

Example:

```jsx
<Avatar
  src={doctor.photoUrl}
  name={doctor.name}
  width={100}
  className="size-12 rounded-full"
/>
```

`width = 200` and `className = ''` are JavaScript destructuring defaults, not React-specific syntax.

## 4. Remember which image URL failed

```jsx
const [failedSrc, setFailedSrc] = useState(null);
```

Initially, `failedSrc` is `null` because no image failure has been recorded. When an image fails, the component saves its URL in state. Updating state makes React render the component again.

1. The component renders with `failedSrc` set to `null`.
2. The browser tries to load the image.
3. If loading fails, `onError` stores the URL with `setFailedSrc(src)`.
4. React re-renders with the updated state.
5. The component displays the initials fallback.

## 5. Decide whether to show initials or an image

```jsx
if (!src || failedSrc === src) {
  // render initials fallback
}
```

`!src` is true when the URL is missing or empty. `failedSrc === src` is true when this exact URL has already failed. Saving the URL instead of a boolean matters if the parent later supplies a new URL: the component can try the new image while remembering that the old one failed.

## 6. Render the accessible initials fallback

```jsx
return (
  <div
    role="img"
    aria-label={name}
    className={`flex items-center justify-center bg-brand/15 font-serif font-bold text-brand ${className}`}
  >
    {initials(name)}
  </div>
);
```

- `role="img"` tells assistive technology that this `div` represents an image-like item.
- `aria-label={name}` supplies a text label for the fallback.
- The Tailwind classes center and style the initials.
- `${className}` lets the parent set the avatar's size and shape.
- `{initials(name)}` calls the helper and renders its result.

## 7. Render the image and handle failures

```jsx
return (
  <img
    src={optimizeImage(src, width)}
    alt={name}
    loading="lazy"
    onError={() => setFailedSrc(src)}
    className={`object-cover ${className}`}
  />
);
```

`optimizeImage(src, width)` is a project helper that can request an appropriately sized Cloudinary image. Resizing an image for its display size can avoid downloading a much larger original; the exact transformation depends on the helper and Cloudinary configuration.

`alt={name}` supplies alternative text. `loading="lazy"` lets the browser defer loading images until they are near the visible area, which can help pages with many doctor cards.

The browser fires `onError` if the image cannot load, for example if the URL returns an error, the network request fails, or the file is invalid. The handler saves the source:

```jsx
onError={() => setFailedSrc(src)}
```

On the next render, `failedSrc === src` is true, so the component returns the initials instead of the broken image.

## 8. Complete component example

```jsx
import { useState } from 'react';

function initials(name = '') {
  const words = name
    .replace(/^dr\.?\s+/i, '')
    .trim()
    .split(/\s+/)
    .filter(Boolean);

  return (
    (words[0]?.[0] || 'D') +
    (words[1]?.[0] || '')
  ).toUpperCase();
}

export function Avatar({ src, name, width = 200, className = '' }) {
  const [failedSrc, setFailedSrc] = useState(null);

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

In the project, `optimizeImage` must also be imported or defined in this module.

## 9. Check your understanding

1. What is the initial value of `failedSrc`? **`null`; no failed URL has been recorded.**
2. What happens after an image fails? **`onError` updates state, React re-renders, and the fallback is shown.**
3. Why compare `failedSrc === src`? **A new URL can still be tried after an older URL failed.**
4. What does `loading="lazy"` do? **It lets the browser defer off-screen image loading.**

## Key takeaways

This component combines string processing, array methods, optional chaining, props, state, conditional rendering, event handling, accessibility, and image optimization. The central idea is that an image failure is represented in React state, so the UI can respond predictably instead of showing a broken image.
