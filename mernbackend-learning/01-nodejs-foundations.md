# 01. Node.js foundations

## What Node.js does

Node.js runs JavaScript outside the browser. It provides APIs for files, networking, environment variables, and processes. Express is a library that runs on Node; it is not the runtime itself.

Node handles many I/O tasks asynchronously. While a database or network operation is waiting, the event loop can process other work. Long CPU-bound JavaScript still blocks that event loop, so expensive work may need a worker or a separate service.

## Modules and startup

Choose either CommonJS (`require`) or ES modules (`import`) consistently. With ES modules, set `"type": "module"` in `package.json` or use `.mjs` files.

```js
// math.js
export function add(a, b) {
  return a + b;
}

// app.js
import { add } from './math.js';
console.log(add(2, 3));
```

## Async work

Use `async`/`await` to make promise-based operations easier to follow. Handle failures at a boundary that can return a useful response or stop startup safely.

```js
try {
  const result = await saveRecord(input);
  console.log(result);
} catch (error) {
  console.error('Could not save record', error);
}
```

Avoid blocking calls such as synchronous file reads in a request handler. Do not start database-dependent requests until the database connection is ready.

## Configuration

Read configuration from environment variables, validate required values during startup, and keep `.env` files out of Git. Use a package such as `dotenv` for local development; production platforms usually inject environment variables directly.

## Practice

Create a small Node script that reads a configuration value, awaits a promise, and handles a rejected promise. Explain which parts use the event loop and which work would block it.
