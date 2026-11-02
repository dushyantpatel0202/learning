# 11. Data integrity and concurrent requests

An API can be correct for one request at a time and still corrupt data when two requests overlap. This matters for limited appointment slots, queue numbers, inventory, and duplicate submissions.

## Why check-then-write can fail

Suppose two requests both check that a time slot is free. Both see it as free before either inserts a booking, so both may succeed. A check in JavaScript does not reserve the slot.

Prefer a database-enforced rule where possible. For example, a unique compound index on the doctor and slot can prevent duplicate bookings. Catch the duplicate-key error and return a conflict response. First verify existing records and decide whether cancelled appointments should still reserve the slot; the index must reflect the real business rule.

```js
appointmentSchema.index(
  { doctor: 1, appointmentDate: 1, timeSlot: 1 },
  { unique: true }
);
```

For counters, use an atomic database update rather than reading a value, incrementing it in application code, and writing it back. When a workflow must update multiple documents together, consider a MongoDB transaction, which requires a deployment configuration that supports transactions. Keep transactions short and handle transient failures appropriately.

## Idempotency

Clients may retry after timeouts even when the first request succeeded. For operations such as payments or booking creation, an idempotency key can let the server recognize and return the original result instead of creating a duplicate. Define key scope, expiry, and storage behavior explicitly.

## Practice

Model two simultaneous requests for the same doctor and time. Identify the database constraint that prevents a double booking, the API response for the losing request, and how cancellation affects slot availability.
