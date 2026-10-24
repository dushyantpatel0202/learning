# Module 10: Frontend Performance, Code Splitting & Optimization

---

## 1. 💡 Mental Model: The Cost of JavaScript

A 2MB JavaScript bundle doesn't just take time to download over 4G mobile networks—it takes mobile CPUs hundreds of milliseconds to parse, compile, and execute before the user can interact with the page.

### Core Web Vitals to Know for Interviews:
1. **LCP (Largest Contentful Paint)**: How fast the main content (e.g. hero banner, doctor photo) renders. Target: $< 2.5\text{s}$.
2. **INP (Interaction to Next Paint)** / **FID**: How responsive the page is when a user taps a button. Target: $< 200\text{ms}$.
3. **CLS (Cumulative Layout Shift)**: Does the page jump around unexpectedly while images and fonts load? Target: $< 0.1$.

---

## 2. ⚡ Route-Level Code Splitting (`React.lazy` + `Suspense`)

Currently in `frontend/src/App.jsx`, all pages (`Home`, `Login`, `Register`, `DoctorDashboard`, `PatientAppointments`) are imported synchronously at the top of the file:
```javascript
// Eager imports: The browser downloads the entire Doctor Dashboard code
// even if a regular patient just wants to view the homepage!
import Home from './pages/Home';
import DoctorDashboard from './pages/DoctorDashboard';
```

### The Production Optimization: Dynamic Imports
With `React.lazy()`, each page is broken into a separate chunk and only downloaded when the user actually navigates to that route:

```jsx
import { lazy, Suspense } from 'react';
import { Routes, Route } from 'react-router-dom';
import Navbar from './components/Navbar';

// Lazy-loaded route chunks
const Home = lazy(() => import('./pages/Home'));
const DoctorProfile = lazy(() => import('./pages/DoctorProfile'));
const DoctorDashboard = lazy(() => import('./pages/DoctorDashboard'));
const PatientAppointments = lazy(() => import('./pages/PatientAppointments'));

export default function App() {
  return (
    <>
      <Navbar />
      {/* Suspense displays a fallback UI while the chunk downloads over the network */}
      <Suspense fallback={<div className="p-8 text-center">Loading page...</div>}>
        <Routes>
          <Route path="/" element={<Home />} />
          <Route path="/doctor/:id" element={<DoctorProfile />} />
          <Route path="/doctor-dashboard" element={<DoctorDashboard />} />
          <Route path="/my-appointments" element={<PatientAppointments />} />
        </Routes>
      </Suspense>
    </>
  );
}
```

---

## 3. 🖼️ Production Image Optimization: `frontend/src/utils/image.js`

Images account for over 60% of average web page weight. Our application applies two industry-standard optimizations:

### 1. Browser-Side Image Resizing Before Upload
In `image.js`, we use the HTML5 Canvas API and `createImageBitmap` to downscale huge phone camera pictures (e.g. 10MB) to $< 300\text{KB}$ before sending them to the cloud:

```javascript
// frontend/src/utils/image.js
export async function resizeImage(file, maxSize = 1280, quality = 0.85) {
  // Respects EXIF orientation so phone camera shots don't rotate sideways
  const bitmap = await createImageBitmap(file);

  const scale = Math.min(1, maxSize / Math.max(bitmap.width, bitmap.height));
  const width = Math.round(bitmap.width * scale);
  const height = Math.round(bitmap.height * scale);

  const canvas = document.createElement('canvas');
  canvas.width = width;
  canvas.height = height;
  const ctx = canvas.getContext('2d');
  ctx.fillStyle = '#fff'; // Prevents transparent PNGs from turning black
  ctx.fillRect(0, 0, width, height);
  ctx.drawImage(bitmap, 0, 0, width, height);
  bitmap.close?.();

  return new Promise((resolve, reject) => {
    canvas.toBlob(blob => blob ? resolve(blob) : reject(), 'image/jpeg', quality);
  });
}
```

### 2. On-The-Fly Cloud CDN Dynamic Sizing
When rendering avatars, we don't serve the original high-res picture. We inject Cloudinary URL parameters to deliver modern WebP/AVIF formats at the exact pixel size needed:

```javascript
export function optimizeImage(url, size) {
  if (!url || !url.includes('//res.cloudinary.com/')) return url;
  // c_fill: crop to fill
  // g_face: auto-center on the human doctor's face!
  // q_auto: automatic intelligent compression
  // f_auto: deliver WebP or AVIF based on browser support
  return url.replace('/image/upload/', `/image/upload/c_fill,g_face,w_${size},h_${size},q_auto,f_auto/`);
}
```

---

## 4. 📜 List Virtualization (Windowing)

### The Problem:
Imagine rendering 5,000 doctors in a state-wide medical directory.
Creating 5,000 React components creates $> 35,000$ Real DOM elements. Scrolling becomes choppy, and browser memory spikes.

### The Solution:
**Virtualization** renders only the 10–15 items currently visible inside the viewport window. As the user scrolls, the items moving offscreen are unmounted and recycled:

```
[ Viewport Window (User sees rows 10-15) ]
---------------------------------------------
Row 10 (Rendered DOM)
Row 11 (Rendered DOM)
Row 12 (Rendered DOM)
Row 13 (Rendered DOM)
Row 14 (Rendered DOM)
---------------------------------------------
(Rows 15-5000 do NOT exist in the browser DOM!)
```

Standard industry libraries: `@tanstack/react-virtual` or `react-window`.

---

## 5. ⚠️ Real-World Production War Story: The Layout Shift Penalty

### The Scenario:
A medical directory loaded doctor cards with avatars:
```jsx
<img src={doctor.photoUrl} alt={doctor.name} />
```
Before the image finished loading from the CDN, the `<img>` tag had a height of $0\text{px}$.
When the image downloaded 400ms later, the card suddenly jumped down by $80\text{px}$.
If a patient was about to click "Book Dr. Verma", the sudden jump caused them to accidentally click on Dr. Sharma!

### Google Web Vitals Penalty:
This caused a massive **CLS (Cumulative Layout Shift)** penalty, hurting the application's SEO rankings on Google Search.

### The Senior Engineer Fix:
Always provide explicit width/height dimensions or an aspect-ratio container in CSS:
```jsx
// Fixed size avatar container prevents layout shift:
<div className="size-14 shrink-0 rounded-[14px] overflow-hidden bg-brand/10">
  <img 
    src={doctor.photoUrl} 
    alt={doctor.name} 
    width="56" 
    height="56" 
    className="h-full w-full object-cover" 
  />
</div>
```

---

## 6. 🎯 Interview Question Bank: Performance Optimization

### Q1: What is the difference between Code Splitting and Tree Shaking?
> **Answer**:
> - **Tree Shaking**: A dead-code elimination build step (handled by Vite/Rollup/Webpack) that removes unused exports from the final bundle (e.g., importing `{ debounce }` from `lodash-es` instead of bundling the whole library).
> - **Code Splitting**: Splitting a single large JavaScript bundle into smaller chunks that are loaded on-demand (e.g., using `React.lazy()` for different routes).

### Q2: How does `React.Suspense` work under the hood?
> **Answer**:
> When a lazy component (or resource) is not ready, it **throws a Promise**. React catches this thrown Promise at the nearest `<Suspense>` boundary, pauses rendering of that subtree, and renders the `fallback` UI. When the Promise resolves, React re-renders the subtree with the loaded module.

### Q3: When does `useCallback` actually degrade performance?
> **Answer**:
> When applied blindly to simple inline functions inside components that aren't wrapped in `React.memo`. Creating a `useCallback` requires allocating a function, creating an array for dependencies, and executing a dependency comparison on every render. If the child re-renders anyway, `useCallback` simply adds CPU overhead for zero gain.

---

## 7. 💻 Hands-On Mastery Exercise

Refactor `frontend/src/App.jsx` to use `React.lazy` and `<Suspense>` for the `DoctorDashboard` and `DoctorProfile` pages.
