# MERN Backend Learning Roadmap

A practical path for learning the server side of a MERN application: Node.js, Express, MongoDB, Mongoose, authentication, security, and deployment. The examples use modern JavaScript modules and Express 5 style async handlers. Adapt package versions to your project.

## Roadmap

| # | Topic | What you will learn |
| --- | --- | --- |
| 01 | [Node.js foundations](./01-nodejs-foundations.md) | Runtime, modules, async work, environment variables |
| 02 | [Express and REST APIs](./02-express-and-rest-apis.md) | App setup, routes, HTTP methods, status codes |
| 03 | [Middleware and request flow](./03-middleware-and-request-flow.md) | Middleware order, routers, request lifecycle |
| 04 | [MongoDB and Mongoose](./04-mongodb-and-mongoose.md) | Schemas, models, references, indexes |
| 05 | [Validation and error handling](./05-validation-and-errors.md) | Input validation, central errors, consistent responses |
| 06 | [Authentication and authorization](./06-authentication-and-authorization.md) | Identity, sessions, roles, resource ownership |
| 07 | [Security and configuration](./07-security-and-configuration.md) | Secrets, CORS, rate limits, safe data handling |
| 08 | [Backend project structure](./08-project-structure.md) | Organizing routes, controllers, services, and models |
| 09 | [Testing and debugging](./09-testing-and-debugging.md) | API tests, database isolation, troubleshooting |
| 10 | [Deployment and operations](./10-deployment-and-operations.md) | Production configuration, logging, health checks |
| 11 | [Data integrity and concurrent requests](./11-data-integrity-and-concurrency.md) | Atomic updates, duplicate requests, transactions |
| 12 | [File uploads and external services](./12-uploads-and-integrations.md) | Upload validation, object storage, reliable integrations |

## Suggested study loop

For each topic, write a small example, call it with an API client, then explain the request flow from HTTP request through database query to response. Keep secrets out of source control, and verify authorization on the server for every protected resource.

## Capstone

Build a small appointments API with users, doctors, and appointments. Include pagination, validation, ownership checks, useful error responses, and tests. Treat the fee and user identity as server-owned values; never trust them just because a client submitted them.
