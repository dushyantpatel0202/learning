# 09. Testing and debugging

Test the observable API behavior, including successful responses and failure cases. Useful layers include unit tests for isolated business rules, integration tests for routes and database behavior, and a small number of end-to-end flows.

## API test checklist

- Correct status and response shape for valid input.
- Missing, malformed, and out-of-range values are rejected.
- Unauthenticated requests cannot access protected routes.
- A user cannot access another user's record by changing an ID.
- Duplicate and concurrent operations preserve business rules.
- Database failures produce safe errors and useful server logs.

Use a dedicated test database or isolated test database setup. Clean up created records, avoid tests that depend on execution order, and never point destructive test setup at production. Keep time-dependent behavior deterministic with a controllable clock where practical.

## Debugging

Start with the request method, URL, headers, body, status, and response. Then inspect server logs with a request identifier. Avoid printing secrets or full personal records. Reproduce bugs with the smallest request that demonstrates them.

## Practice

Write an integration test for creating an appointment, then add tests for invalid date, missing authentication, and booking another patient's appointment ID.
