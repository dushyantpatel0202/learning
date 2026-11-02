# 10. Deployment and operations

Deployment runs the API in an environment with production configuration, a reachable database, HTTPS termination, and a process manager or hosting platform that restarts failed processes.

## Before release

- Validate environment configuration at startup and keep secrets out of the repository.
- Use a managed database or secured database deployment with network access restricted to the application.
- Configure allowed frontend origins, HTTPS, and production cookie settings.
- Run database migrations or index changes deliberately and review their impact on existing data.
- Use structured logs with request IDs; do not log passwords, tokens, or unnecessary personal data.
- Add a health endpoint that reports application readiness without exposing secrets or internal topology.
- Set sensible timeouts and graceful shutdown behavior.

Monitor errors, latency, database availability, and resource use. Back up important data and periodically confirm that restores work. A deployment is not complete until the running service can be observed and recovered.

## Practice

Write a release checklist for the appointments API. Include configuration, database connectivity, a smoke test, logs, and a rollback or recovery plan.
