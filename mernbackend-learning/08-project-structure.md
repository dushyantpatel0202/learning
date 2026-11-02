# 08. Backend project structure

Organize code around responsibilities so HTTP details, business rules, and persistence are easy to understand. A small project can start simply and split files as repeated logic appears.

```text
src/
  app.js                 # middleware and route mounting
  server.js              # configuration, database connection, listen
  routes/                # URL and method mapping
  controllers/           # HTTP input/output
  services/              # business rules and workflows
  models/                # Mongoose schemas and models
  middleware/             # authentication, validation, errors
  config/                 # validated configuration
```

Routes should define the endpoint and its middleware. Controllers translate the request and response. Services implement reusable business operations. Models define persistence shape and constraints. This is a useful separation, not a rule that every small handler needs its own file.

Keep startup separate from app construction so tests can import the app without opening a network port. Connect to the database before accepting requests that need it, and handle shutdown signals so connections can close cleanly.

## Practice

Take one endpoint and trace its route, validation, controller, service, model query, and response. Move only logic that improves clarity or reuse.
