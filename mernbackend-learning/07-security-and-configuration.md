# 07. Security and configuration

Security is part of every endpoint. Validate input, authenticate callers, authorize each resource operation, and return only fields the caller is allowed to see.

## Practical safeguards

- Keep secrets in environment variables or a secret manager; commit only a `.env.example` with placeholder values.
- Configure CORS with the specific browser origins that need access. CORS is not authentication and does not stop non-browser clients.
- Use HTTPS in deployed environments.
- Apply rate limits to sensitive or expensive endpoints, especially login and search.
- Set appropriate security headers and limit request body size.
- Avoid injection by using parameterized database APIs and allow-listed filters and updates.
- Redact credentials and personal data from logs.
- Keep dependencies updated and review advisories through the package manager.

Do not enable permissive settings such as wildcard credentialed CORS as a shortcut. Understand the threat each control addresses and configure it for the app's actual deployment.

## Configuration validation

Fail early at startup if required values such as the database URI or signing key are missing. Separate development, test, and production configuration, and never use production data casually in local tests.

## Practice

Review one endpoint and list its input sources, authentication requirement, authorization rule, data it returns, and abuse controls.
