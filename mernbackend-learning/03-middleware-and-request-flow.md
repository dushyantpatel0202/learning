# 03. Middleware and request flow

Middleware is a function that can inspect or change a request, end the response, or pass control to the next middleware. Its registration order matters.

```js
app.use(express.json());          // parse JSON bodies
app.use(requestLogger);           // observe requests
app.use('/api/doctors', doctorRouter); // route matching
app.use(notFoundHandler);         // unmatched routes
app.use(errorHandler);            // errors passed to next(error)
```

A typical request passes through parsing, logging, authentication (when needed), route-specific validation, the route handler, and response serialization. Middleware that sends a response must not also call `next()` for the same request.

## Routers

Routers group related endpoints and mount under a prefix:

```js
import { Router } from 'express';
const router = Router();

router.get('/', listDoctors);
router.get('/:doctorId', getDoctor);
export default router;
```

Mount this with `app.use('/api/doctors', router)`. Keep middleware scoped to the routes where it is needed. For Express 5, rejected promises from async route handlers are forwarded to error handling; check your Express version before relying on that behavior.

## Practice

Add a request logger that records method, path, and response status. Avoid logging passwords, tokens, or other sensitive request values.
