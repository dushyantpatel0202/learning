# Module 07: Forms, Controlled vs Uncontrolled Inputs & File Uploads

---

## 1. 💡 Mental Model: Controlled vs Uncontrolled Inputs

In web forms, DOM input elements natively maintain their own internal state (what you type into `<input>`).
In React, we have two distinct ways of working with forms:

### Controlled Components (The React Way)
React state is the **single source of truth**. Every keystroke updates React state, and React pushes that value back into the input:
```jsx
const [name, setName] = useState('');

<input 
  value={name} 
  onChange={(e) => setName(e.target.value)} 
/>
```
- **Pros**: Instant validation, formatted masking (phone numbers, currency), easy conditional disabling of buttons.
- **Cons**: Causes a re-render on every keystroke (usually negligible, but noticeable on gigantic forms).

### Uncontrolled Components (The DOM Way)
The DOM holds the state. React reads the value only when the form is submitted via a `ref` or `FormData`:
```jsx
const nameRef = useRef(null);

function handleSubmit(e) {
  e.preventDefault();
  console.log(nameRef.current.value);
}

<input ref={nameRef} defaultValue="Dr. Sharma" />
```
- **Pros**: Ultra-fast; zero re-renders while typing.
- **Cons**: Harder to validate dynamically as the user types.

---

## 2. 📝 Handling Complex Multi-Field Forms

Instead of writing 15 separate `useState` hooks for 15 inputs, use a **single state object** with dynamic keys:

```jsx
const [formData, setFormData] = useState({
  name: '',
  specialization: '',
  experienceYears: 0,
  clinicName: ''
});

function handleChange(e) {
  const { name, value, type } = e.target;
  
  setFormData(prev => ({
    ...prev,
    [name]: type === 'number' ? Number(value) : value
  }));
}

return (
  <form>
    <input name="name" value={formData.name} onChange={handleChange} />
    <input name="specialization" value={formData.specialization} onChange={handleChange} />
    <input name="experienceYears" type="number" value={formData.experienceYears} onChange={handleChange} />
  </form>
);
```

---

## 3. 🖼️ Production File Upload Architecture: `PhotoUploader.jsx`

Handling file uploads in production is a critical interview topic. 
Uploading 10MB images directly to your Node.js server crashes memory and bottlenecks bandwidth.
The industry-standard pattern is **Direct Signed Cloud Uploads**:

```
[Browser] ──────── 1. Request signed upload URL ───────► [Node/Express Server]
[Browser] ◄─────── 2. Returns signature + timestamp ───── [Node/Express Server]
[Browser] ──────── 3. Upload binary directly to Cloudinary / AWS S3 ──────► [Cloudinary]
[Browser] ◄─────── 4. Returns permanent secure image URL ────────────── [Cloudinary]
[Browser] ──────── 5. Save photo URL into MongoDB ────────────────────► [Node/Express Server]
```

### Real Project Code Walkthrough: `frontend/src/components/PhotoUploader.jsx`

```jsx
// frontend/src/components/PhotoUploader.jsx
export default function PhotoUploader({ onUploaded }) {
  const galleryInput = useRef(null);
  const [busy, setBusy] = useState(false);
  const [progress, setProgress] = useState(0);

  async function handleFile(e) {
    const file = e.target.files?.[0];
    e.target.value = ''; // Reset so re-selecting same photo triggers onChange!
    if (!file) return;

    // 1. Client-side MIME validation:
    if (!file.type.startsWith('image/')) {
      alert('Please upload a valid JPG or PNG image.');
      return;
    }

    // 2. Client-side file size gate (15MB):
    if (file.size > 15 * 1024 * 1024) {
      alert('File is too large. Maximum size is 15MB.');
      return;
    }

    setBusy(true);
    try {
      // 3. Client-side Canvas downscaling (e.g. 6MB -> 300KB)
      const compressedBlob = await resizeImage(file);

      // 4. Request cryptographic upload signature from our backend
      const { data: sig } = await api.post('/uploads/signature');

      // 5. Build FormData for Cloudinary
      const body = new FormData();
      body.append('file', compressedBlob, 'profile.jpg');
      body.append('api_key', sig.apiKey);
      body.append('timestamp', sig.timestamp);
      body.append('signature', sig.signature);
      body.append('folder', sig.folder);

      // 6. Direct upload to Cloudinary with progress tracking
      const res = await axios.post(
        `https://api.cloudinary.com/v1_1/${sig.cloudName}/image/upload`,
        body,
        {
          onUploadProgress: (evt) => {
            if (evt.total) setProgress(Math.round((evt.loaded / evt.total) * 100));
          }
        }
      );

      // 7. Pass the resulting URL back to parent form
      onUploaded(res.data.secure_url);
    } catch (err) {
      alert('Upload failed: ' + err.message);
    } finally {
      setBusy(false);
    }
  }

  return (
    <div>
      <input 
        ref={galleryInput} 
        type="file" 
        accept="image/*" 
        className="hidden" 
        onChange={handleFile} 
      />
      <button type="button" onClick={() => galleryInput.current.click()}>
        Choose Doctor Photo
      </button>
      {busy && <div className="text-sm">Uploading: {progress}%</div>}
    </div>
  );
}
```

> [!CAUTION]
> **Security Rule**:
> Notice that the Cloudinary upload uses plain `axios.post(...)`, **NOT** our pre-configured `api.post(...)`. Why? Because `api.js` attaches the user's JWT Authorization Bearer token to all headers. Sending our app's JWT token to a third-party server (Cloudinary) would leak user credentials!

---

## 4. ⚠️ Real-World Production War Story: The Frozen Mobile File Picker

### The Bug:
Doctors using mobile phones tapped "Upload Profile Picture", chose a photo, realized it was the wrong photo, canceled, and tapped again to select the exact same photo. **Nothing happened!** The file picker refused to open or fire.

### Why It Happened:
In HTML, `<input type="file" onChange={handleFile} />` only fires the `change` event when `input.value` changes. When a user selects the same file again, the value hasn't changed, so React doesn't trigger `onChange`!

### The Senior Engineer Fix:
Notice line 31 in `PhotoUploader.jsx`:
```javascript
const file = e.target.files?.[0];
e.target.value = ''; // Instantly reset the input value so selecting the same file triggers onChange!
```

---

## 5. 🎯 Interview Question Bank: Forms & Uploads

### Q1: What is the difference between `defaultValue` and `value` in React inputs?
> **Answer**:
> - `value` sets the input to **Controlled mode**. The input's displayed value is bound strictly to React state. If you provide `value` without an `onChange` handler, the input becomes read-only in the browser.
> - `defaultValue` sets the input to **Uncontrolled mode**. It defines only the initial value on mount; subsequent keystrokes are managed entirely by the browser DOM.

### Q2: Why should file uploads avoid going directly through your Express backend server?
> **Answer**:
> Routing large binary files (images, PDFs, videos) through Node.js consumes server CPU, fills temporary server memory/disk buffers, and ties up Express worker threads that should be answering fast JSON API calls.
> Generating a signed URL or signature from the server and having the browser stream the binary file directly to cloud object storage (Cloudinary, AWS S3, Google Cloud Storage) is infinitely scalable.

### Q3: How do you prevent a form submission from refreshing the browser page?
> **Answer**:
> In the form's `onSubmit` handler, call `e.preventDefault()`. In HTML standards, submitting a form issues a POST navigation that refreshes the page and drops client-side state.

---

## 6. 💻 Hands-On Mastery Exercise

Look at `DoctorDashboard.jsx`. Add client-side validation before saving the profile:
1. `name` must be at least 3 characters.
2. `experienceYears` must be a positive integer $\ge 0$.
3. If validation fails, display inline field error messages in red without making the API call.
