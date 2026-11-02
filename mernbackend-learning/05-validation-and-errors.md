# 05. Validation and error handling

Treat every request value as untrusted, including route parameters, query parameters, headers, and JSON bodies. Validate shape, type, length, allowed values, and business rules before performing writes. Schema validation in Mongoose is useful but does not replace API validation or authorization.

## Allow-list input

Build an object from fields the client may change instead of copying the whole body:

```js
const updates = {};
if (typeof req.body.name === 'string') updates.name = req.body.name.trim();
if (typeof req.body.city === 'string') updates.city = req.body.city.trim();
```

In a larger app, use a request validation library and keep validation schemas beside their routes. Reject invalid input with `400 Bad Request` or `422 Unprocessable Content`, choosing one convention and applying it consistently.

## Central error handler

Use one final error middleware to map known errors to safe HTTP responses. Log unexpected errors on the server, but do not expose stack traces or database connection details to clients.

```js
app.use((err, _req, res, _next) => {
  const status = err.statusCode ?? 500;
  res.status(status).json({ message: status === 500 ? 'Internal server error' : err.message });
});
```

Use `404` for a missing resource, `401` when authentication is missing or invalid, `403` when an authenticated user lacks permission, and `409` for a state conflict where appropriate.

## Practice

Design validation for booking an appointment: valid IDs, a future date, an allowed time, and a doctor who is accepting bookings. Identify which checks must be enforced atomically or by a database constraint to handle simultaneous requests.
