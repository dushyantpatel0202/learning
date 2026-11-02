# 12. File uploads and external services

File uploads and third-party APIs add failure modes beyond ordinary JSON requests. Keep uploaded files in object storage or a dedicated media service rather than growing the application database with large binary data in routine cases. Store only the file metadata and trusted storage key needed by the application.

## Upload handling

- Enforce size limits at the proxy and application layers.
- Allow-list expected file types and inspect file contents where practical; a client-provided MIME type alone can be spoofed.
- Generate storage names on the server instead of trusting user-supplied paths or filenames.
- Require authentication and authorization for private uploads and downloads.
- Consider malware scanning for files that will be shared or processed.
- Avoid serving user uploads from a location where they can execute as application code.

For private files, use authenticated download endpoints or short-lived signed URLs. Do not expose storage credentials or unrestricted bucket permissions to the browser.

## External services

Treat email, payment, maps, and identity providers as unreliable dependencies. Set timeouts, handle errors, avoid logging credentials, and define what the user sees when the service is unavailable. Verify webhooks using the provider's signature mechanism and make webhook processing idempotent because providers may retry delivery.

For work that can happen after the API response, such as sending a confirmation email, a background queue can improve resilience. Persist the intent to perform the work and make the worker safe to retry; do not assume an email or payment call succeeds just because the request was accepted.

## Practice

Design a doctor profile photo upload flow. Specify allowed formats and size, storage location, who can replace the image, how old images are cleaned up, and what happens if storage is unavailable.
