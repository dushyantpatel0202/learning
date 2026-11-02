# 02. Express and REST APIs

Express maps incoming HTTP requests to handlers. A REST-style API uses resource-oriented URLs and HTTP methods to describe the requested operation.

| Intent | Method and example | Typical success |
| --- | --- | --- |
| List resources | `GET /api/doctors` | `200 OK` |
| Read one | `GET /api/doctors/:id` | `200 OK` |
| Create | `POST /api/appointments` | `201 Created` |
| Replace or update | `PUT /api/doctors/:id` | `200 OK` |
| Partial update | `PATCH /api/doctors/:id` | `200 OK` |
| Delete | `DELETE /api/appointments/:id` | `204 No Content` |

## Minimal application

```js
import express from 'express';

const app = express();
app.use(express.json());

app.get('/api/health', (_req, res) => {
  res.status(200).json({ status: 'ok' });
});

app.listen(process.env.PORT ?? 3000);
```

Register body parsing before routes that read `req.body`. Return JSON consistently, choose status codes that describe the result, and avoid returning database internals or stack traces to clients.

## Resource design

Prefer nouns for resources, such as `/api/appointments`. Use query parameters for filtering and pagination, for example `GET /api/doctors?city=Pune&page=2&limit=20`. Validate and cap pagination limits so clients cannot request unbounded results.

## Practice

Add routes to list and fetch doctors. Decide what response should be returned for an unknown ID, malformed ID, and database failure. Keep route handlers focused; move repeated business logic into a service as the API grows.
