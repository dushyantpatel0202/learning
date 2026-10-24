# Lesson 5: POST /api/uploads/signature (Secure Direct Cloud Uploads)

In **Lesson 4**, we learned how patients request live tokens.  
In **Lesson 5**, we complete the POST request roadmap with an essential production pattern: **Offloading Heavy File Uploads via Server-Signed Signatures**.

When doctors upload their profile photo in [PhotoUploader.jsx](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/frontend/src/components/PhotoUploader.jsx), we never stream high-resolution image binaries directly through our Node.js server. Streaming multi-megabyte images through Express would block server event loops, inflate server memory, and spike bandwidth costs.

Instead, our frontend asks our backend for a cryptographically signed permission slip via **`POST /api/uploads/signature`**, and then uploads the photo straight to Cloudinary.

---

## Roadmap of POST Requests
1. ✅ **POST /api/auth/session** — Firebase token verification & JWT session creation (`AuthContext.jsx` & `authController.js`)
2. ✅ **POST /api/doctors** — Creating a doctor profile & doctorCode generation (`DoctorDashboard.jsx` & `doctorController.js`)
3. ✅ **POST /api/appointments** — Booking an appointment with consultation slot & fees (`BookingModal.jsx` & `appointmentController.js`)
4. ✅ **POST /api/tokens/doctor/:doctorId** — Generating a live queue token for today (`TokenDisplay.jsx` & `tokenController.js`)
5. 🎯 **POST /api/uploads/signature** — Minting a secure Cloudinary upload signature (`PhotoUploader.jsx` & `uploadController.js`)

---

## Part 1 — Backend: Route Definition & RBAC

File: [backend/routes/uploads.js](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/backend/routes/uploads.js)

```javascript
const express = require('express');
const asyncHandler = require('../middleware/asyncHandler');
const { protect, doctorOnly } = require('../middleware/auth');
const { getProfilePhotoSignature } = require('../controllers/uploadController');

const router = express.Router();

router.post('/signature', protect, doctorOnly, asyncHandler(getProfilePhotoSignature));

module.exports = router;
```

### Line-by-Line Breakdown:
- `router.post('/signature', ...)`: Accepts POST requests to generate signed upload tickets.
- `protect`: Verifies the doctor's JWT session.
- `doctorOnly`: Restricts upload permissions. Ordinary patients cannot generate upload signatures, protecting Cloudinary storage quota from unauthorized abuse.

---

## Part 2 — Backend: Controller Logic & Cryptographic Signing

File: [backend/controllers/uploadController.js](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/backend/controllers/uploadController.js)

```javascript
const { cloudinary, getCredentials, isConfigured } = require('../config/cloudinary');

async function getProfilePhotoSignature(req, res) {
  if (!isConfigured()) {
    return res.status(503).json({
      message: 'Photo upload is not set up on the server yet (missing Cloudinary settings).',
    });
  }

  const { cloudName, apiKey, apiSecret } = getCredentials();

  const params = {
    timestamp: Math.round(Date.now() / 1000), // Expire after ~1 hour
    folder: 'apna-doctor/doctors',
    // One fixed file per doctor: doctor_<userId>
    public_id: `doctor_${req.user.id}`,
    overwrite: true,
    // Transformations applied directly on upload
    transformation: 'c_fill,g_face,w_600,h_600,q_auto',
  };

  // Sign using our secret API key
  const signature = cloudinary.utils.api_sign_request(params, apiSecret);

  // Send signed parameters back to browser
  res.json({ cloudName, apiKey, signature, ...params });
}
```

### Granular Line-by-Line Breakdown:
1. `public_id: doctor_${req.user.id}`: **Crucial Security Constraint**. The backend locks the uploaded file's destination name strictly to the authenticated user's ID (`req.user.id`). The client cannot choose where the file is stored.
2. `overwrite: true`: Automatically replaces previous profile images for this doctor. No orphan images accumulate in the cloud, keeping free tier storage clean.
3. `transformation: 'c_fill,g_face,w_600,h_600,q_auto'`: Direct Cloudinary AI transformation instructions: detects the doctor's face (`g_face`), crops into a 600x600 square (`c_fill`), and automatically compresses quality (`q_auto`).
4. `cloudinary.utils.api_sign_request(params, apiSecret)`: Uses SHA-1 / SHA-256 HMAC hashing with our private `apiSecret`. Because `apiSecret` never leaves the server, malicious clients cannot tamper with the parameters.

---

## Part 3 — Frontend: Client Resizing & Two-Step Upload

File: [frontend/src/components/PhotoUploader.jsx](file:///D:/Doctor-appointment-app/version%203/Doctor-appointment-app/frontend/src/components/PhotoUploader.jsx)

```javascript
// Step 1: Pre-shrink photo on client before sending to save network bandwidth
const photo = await resizeImage(file);

// Step 2: Request signed authorization from our server
const { data: sig } = await api.post('/uploads/signature');

// Step 3: Bundle exact parameters signed by the server
const body = new FormData();
body.append('file', photo, 'profile.jpg');
body.append('api_key', sig.apiKey);
body.append('timestamp', sig.timestamp);
body.append('signature', sig.signature);
body.append('folder', sig.folder);
body.append('public_id', sig.public_id);
body.append('overwrite', sig.overwrite);
body.append('transformation', sig.transformation);

// Step 4: Upload directly to Cloudinary using plain axios (NO app auth token attached!)
const res = await axios.post(`https://api.cloudinary.com/v1_1/${sig.cloudName}/image/upload`, body, {
  onUploadProgress: (ev) => ev.total && setProgress(Math.round((ev.loaded * 100) / ev.total)),
});

// Step 5: Pass the resulting secure_url to parent component
await onUploaded(res.data.secure_url);
```

### Key Architectural Principles:
1. **Client-side compression**: `resizeImage(file)` reduces an 8MB mobile camera photo to ~300KB in browser canvas before uploading, ensuring fast transfers over mobile networks.
2. **Plain `axios` vs `api` helper**: Notice that Step 4 uses raw `axios.post` instead of our custom `api` instance. Our `api` helper automatically injects the user's secret JWT token into request headers. **A user's JWT must NEVER be sent to third-party endpoints like Cloudinary!**
3. **Progress feedback**: `onUploadProgress` computes real-time percentage (`0%` to `100%`) for a smooth animated progress bar.

---

## Part 4 — The Full End-to-End Data Flow

```
[Doctor in DoctorDashboard]
       │
       ▼ 1. Selects image from phone or computer
[PhotoUploader.jsx]
       │
       ▼ 2. Browser resizes image to max bounds (Canvas API)
       │
       ▼ 3. POST /api/uploads/signature
            Headers: Authorization: Bearer <doctor_token>
       │
       ▼ 4. Express Server (uploads.js -> uploadController.js):
            - Verifies protect & doctorOnly
            - Assembles parameters: folder, public_id="doctor_123", transformation
            - Generates cryptographic HMAC SHA signature with CLOUDINARY_API_SECRET
       │
       ▼ 5. Express Responds:
            HTTP 200 OK
            Body: { cloudName: "...", apiKey: "...", signature: "a89fc...", ... }
       │
       ▼ 6. Browser receives signature
       │
       ▼ 7. Browser dispatches multipart/form-data POST directly to Cloudinary:
            POST https://api.cloudinary.com/v1_1/<cloudName>/image/upload
       │
       ▼ 8. Cloudinary verifies signature against parameters:
            - Processes transformation (crops to face, 600x600)
            - Saves file to apna-doctor/doctors/doctor_123
       │
       ▼ 9. Cloudinary returns JSON:
            { secure_url: "https://res.cloudinary.com/.../doctor_123.jpg" }
       │
       ▼ 10. PhotoUploader triggers onUploaded(secure_url)
             -> DoctorDashboard updates avatar image in UI
```

---

## Part 5 — Quick Self-Test

1. **Why don't we upload the image directly to our Express server?**  
   Streaming large files through Node.js monopolizes CPU and RAM, saturates server network bandwidth, and increases hosting costs. Direct uploads let Cloudinary's global CDN handle binary ingestion while our server only processes a lightweight JSON handshake.

2. **Why does Step 4 use plain `axios` instead of our `api` utility?**  
   Our `api` utility attaches the user's secret JWT token to the `Authorization` header. Sending our application's JWT to Cloudinary or any third-party domain would leak user session tokens.

3. **What happens if someone attempts to change `public_id` to overwrite another doctor's photo?**  
   Cloudinary validates the `signature` against all received parameters. If any parameter (like `public_id` or `folder`) is altered by the client, the HMAC signature verification fails immediately, and Cloudinary rejects the upload with an error.
