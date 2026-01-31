---
name: fastify-best-practices
description: Fastify framework best practices with deep focus on plugin architecture, encapsulation, and composition patterns. Use when building, reviewing, or refactoring Fastify applications. Covers plugin registration, fastify-plugin usage, decorators, hooks, project structure, and common anti-patterns.
metadata:
  author: timmywheels
  version: "1.0"
---

# Fastify Best Practices

This skill provides authoritative guidance on Fastify best practices with a primary focus on the plugin architecture. The plugin system is the core architectural pattern in Fastify—understanding it deeply is essential for building maintainable, scalable Node.js backends.

## The Plugin Architecture: Fastify's Core Design Pattern

Fastify is built around a **plugin-based architecture**. Everything in Fastify is a plugin: routes, decorators, hooks, even other plugins. This is not optional—it's the fundamental mental model for building Fastify applications correctly.

### Why Plugins Matter

```
┌─────────────────────────────────────────────────────────┐
│                    Root Fastify Instance                │
│                                                         │
│  ┌─────────────────┐  ┌─────────────────┐              │
│  │  Plugin A       │  │  Plugin B       │              │
│  │  (encapsulated) │  │  (encapsulated) │              │
│  │                 │  │                 │              │
│  │  ┌───────────┐  │  │  ┌───────────┐  │              │
│  │  │ Child     │  │  │  │ Child     │  │              │
│  │  │ Plugin    │  │  │  │ Plugin    │  │              │
│  │  └───────────┘  │  │  └───────────┘  │              │
│  └─────────────────┘  └─────────────────┘              │
└─────────────────────────────────────────────────────────┘
```

Plugins provide:

- **Encapsulation**: Each plugin gets its own isolated context
- **Composition**: Build complex apps from simple, testable pieces
- **Reusability**: Share plugins across projects via npm
- **Lazy loading**: Plugins are loaded asynchronously in the correct order

---

## Encapsulation: The Most Important Concept

**Encapsulation is what makes Fastify's plugin system powerful.** When you register a plugin, Fastify creates a new context (a child instance) that inherits from the parent but isolates its own additions.

### What Gets Encapsulated

```typescript
// Everything added inside a plugin is encapsulated by default:
// - Decorators (decorate, decorateRequest, decorateReply)
// - Hooks
// - Routes
// - Content type parsers
// - Error handlers
```

### Correct: Encapsulated Plugin

```typescript
import { FastifyPluginAsync } from "fastify";
import fp from "fastify-plugin";

// This plugin's decorator is ONLY visible to itself and its children
const usersPlugin: FastifyPluginAsync = async (fastify) => {
  // This decorator is encapsulated—parent and sibling plugins cannot see it
  fastify.decorate("userService", {
    findById: async (id: string) => {
      /* ... */
    },
  });

  fastify.get("/users/:id", async (request, reply) => {
    const user = await fastify.userService.findById(request.params.id);
    return user;
  });
};

export default usersPlugin;
```

### Breaking Encapsulation with fastify-plugin

Use `fastify-plugin` (fp) **only when you intentionally want to expose** decorators, hooks, or other additions to the parent scope:

```typescript
import fp from "fastify-plugin";

// This decorator WILL be visible to the parent and all siblings
const databasePlugin: FastifyPluginAsync = async (fastify) => {
  const db = await connectToDatabase();

  fastify.decorate("db", db);

  fastify.addHook("onClose", async () => {
    await db.close();
  });
};

// fp() breaks encapsulation—the 'db' decorator bubbles up
export default fp(databasePlugin, {
  name: "database-plugin",
  // Optionally specify dependencies
  dependencies: [],
});
```

### When to Use `fastify-plugin` (fp)

| Use `fp()`                   | Don't use `fp()`                |
| ---------------------------- | ------------------------------- |
| Database connections         | Feature modules (users, orders) |
| Authentication decorators    | Route groups                    |
| Shared utilities             | Domain-specific logic           |
| Config that all plugins need | Anything feature-specific       |

**Rule of thumb**: If you're building infrastructure that the whole app needs, use `fp()`. If you're building a feature, don't.

---

## Plugin Registration Patterns

### The Async Plugin Signature

Always use async plugins—they're cleaner and better supported:

```typescript
import { FastifyPluginAsync } from "fastify";

const myPlugin: FastifyPluginAsync<MyPluginOptions> = async (
  fastify,
  options,
) => {
  // Your plugin code here
  // No need for done() callback—just use async/await
};

export default myPlugin;
```

### Registration Order Matters

Plugins are loaded in registration order, but Fastify handles async resolution:

```typescript
// app.ts
import Fastify from "fastify";

const app = Fastify({ logger: true });

// 1. Infrastructure plugins first (these use fp() to share decorators)
await app.register(import("./plugins/config"));
await app.register(import("./plugins/database"));
await app.register(import("./plugins/auth"));

// 2. Feature plugins second (these are encapsulated)
await app.register(import("./features/users"), { prefix: "/api/users" });
await app.register(import("./features/orders"), { prefix: "/api/orders" });

// 3. Start the server
await app.listen({ port: 3000 });
```

### Using Prefixes for Route Namespacing

```typescript
// Prefix all routes in this plugin with /api/v1/users
await app.register(import("./features/users"), { prefix: "/api/v1/users" });

// Inside the plugin, routes are relative to the prefix
const usersPlugin: FastifyPluginAsync = async (fastify) => {
  fastify.get("/", async () => {
    /* GET /api/v1/users */
  });
  fastify.get("/:id", async () => {
    /* GET /api/v1/users/:id */
  });
  fastify.post("/", async () => {
    /* POST /api/v1/users */
  });
};
```

---

## Decorators: Extending Fastify Correctly

Decorators are how you add properties/methods to Fastify instances, requests, or replies.

### Types of Decorators

```typescript
// Add to the Fastify instance
fastify.decorate("myUtil", () => {
  /* ... */
});

// Add to every request object
fastify.decorateRequest("user", null); // null is the initial value

// Add to every reply object
fastify.decorateReply("sendSuccess", function (data) {
  return this.send({ success: true, data });
});
```

### Critical: Declare Decorator Shapes for Performance

Fastify optimizes object shapes. **Always provide the initial type structure**:

```typescript
// WRONG: Don't do this (breaks V8 optimization)
fastify.decorateRequest("user"); // undefined shape

// CORRECT: Provide initial value matching the shape
fastify.decorateRequest("user", null); // Will be User | null

// For objects, provide a getter if the initial value is expensive
fastify.decorateRequest("parsedBody", {
  getter() {
    // Lazy initialization
    return JSON.parse(this.body);
  },
});
```

### TypeScript: Augmenting Fastify Types

```typescript
// types.d.ts or at the top of your plugin
declare module "fastify" {
  interface FastifyInstance {
    db: DatabaseClient;
    config: AppConfig;
  }

  interface FastifyRequest {
    user: User | null;
  }

  interface FastifyReply {
    sendSuccess: (data: unknown) => void;
  }
}
```

---

## Hooks in the Plugin Context

Hooks registered inside a plugin respect encapsulation:

```typescript
const adminPlugin: FastifyPluginAsync = async (fastify) => {
  // This hook ONLY runs for routes in this plugin and its children
  fastify.addHook("onRequest", async (request, reply) => {
    if (!request.user?.isAdmin) {
      reply.code(403).send({ error: "Admin access required" });
    }
  });

  fastify.get("/admin/dashboard", async () => {
    /* ... */
  });
  fastify.get("/admin/users", async () => {
    /* ... */
  });
};
```

### Hook Execution Order

```
Request Lifecycle:
onRequest → preParsing → preValidation → preHandler → [handler] →
preSerialization → onSend → onResponse

Application Lifecycle:
onReady → onListen → onClose
```

---

## Plugin Dependencies

Declare dependencies to ensure proper loading order:

```typescript
import fp from "fastify-plugin";

const ordersPlugin: FastifyPluginAsync = async (fastify) => {
  // Safe to use fastify.db here—dependency guarantees it exists
  const orders = await fastify.db.query("SELECT * FROM orders");
};

export default fp(ordersPlugin, {
  name: "orders-plugin",
  dependencies: ["database-plugin", "auth-plugin"],
});
```

---

## Project Structure: Organizing Plugins

```
src/
├── app.ts                 # Main app setup, plugin registration
├── plugins/               # Infrastructure plugins (use fp())
│   ├── config.ts          # Environment config
│   ├── database.ts        # Database connection
│   ├── auth.ts            # Authentication decorator
│   └── swagger.ts         # API documentation
├── features/              # Feature plugins (encapsulated)
│   ├── users/
│   │   ├── index.ts       # Plugin entry point
│   │   ├── routes.ts      # Route definitions
│   │   ├── handlers.ts    # Route handlers
│   │   ├── schemas.ts     # JSON schemas
│   │   └── service.ts     # Business logic
│   └── orders/
│       └── ...
└── types/
    └── fastify.d.ts       # Type augmentations
```

### Feature Plugin Structure

```typescript
// features/users/index.ts
import { FastifyPluginAsync } from "fastify";
import { userRoutes } from "./routes";
import { UserService } from "./service";

const usersPlugin: FastifyPluginAsync = async (fastify) => {
  // Create service with injected dependencies
  const userService = new UserService(fastify.db);

  // Decorate for use in handlers (encapsulated to this plugin)
  fastify.decorate("userService", userService);

  // Register routes
  await fastify.register(userRoutes);
};

export default usersPlugin; // NOT wrapped in fp()—stays encapsulated
```

---

## Auto-loading Plugins with @fastify/autoload

For larger applications, use autoload to automatically register plugins:

```typescript
import autoload from "@fastify/autoload";
import { join } from "path";

await app.register(autoload, {
  dir: join(__dirname, "plugins"),
  // These become shared (fp-wrapped)
});

await app.register(autoload, {
  dir: join(__dirname, "features"),
  options: { prefix: "/api" },
  // These stay encapsulated
});
```

---

## Testing Plugins

Plugins are highly testable because of encapsulation:

```typescript
import Fastify from "fastify";
import { test } from "node:test";
import usersPlugin from "../features/users";
import databasePlugin from "../plugins/database";

test("users plugin", async (t) => {
  const app = Fastify();

  // Register only what this plugin needs
  await app.register(databasePlugin);
  await app.register(usersPlugin, { prefix: "/users" });

  await t.test("GET /users/:id returns user", async () => {
    const response = await app.inject({
      method: "GET",
      url: "/users/123",
    });

    assert.strictEqual(response.statusCode, 200);
  });

  await app.close();
});
```

---

## Common Anti-Patterns to Avoid

### 1. Registering Everything at Root Level

```typescript
// WRONG: No encapsulation, everything is global
app.decorate("userService", userService);
app.decorate("orderService", orderService);
app.get("/users", handler);
app.get("/orders", handler);

// CORRECT: Use plugins for organization
await app.register(usersPlugin, { prefix: "/users" });
await app.register(ordersPlugin, { prefix: "/orders" });
```

### 2. Overusing fastify-plugin

```typescript
// WRONG: Breaking encapsulation unnecessarily
export default fp(usersPlugin); // Why expose internal decorators?

// CORRECT: Only use fp() for shared infrastructure
export default usersPlugin;
```

### 3. Synchronous Plugin Pattern

```typescript
// WRONG: Old callback style
const plugin = (fastify, opts, done) => {
  fastify.get("/", handler);
  done();
};

// CORRECT: Modern async style
const plugin: FastifyPluginAsync = async (fastify, opts) => {
  fastify.get("/", handler);
};
```

### 4. Not Declaring Decorator Types

```typescript
// WRONG: Breaks V8 hidden class optimization
fastify.decorateRequest("user");

// CORRECT: Provide initial shape
fastify.decorateRequest("user", null);
```

### 5. Circular Dependencies Between Plugins

```typescript
// WRONG: Plugin A depends on B, B depends on A
// This will fail or cause undefined behavior

// CORRECT: Extract shared logic to a third plugin both can depend on
```

---

## Quick Reference

| Concept                     | Purpose                                            |
| --------------------------- | -------------------------------------------------- |
| `fastify.register()`        | Load a plugin, creating a new encapsulated context |
| `fastify-plugin` (fp)       | Break encapsulation, expose decorators to parent   |
| `fastify.decorate()`        | Add property to Fastify instance                   |
| `fastify.decorateRequest()` | Add property to every request                      |
| `fastify.decorateReply()`   | Add property to every reply                        |
| `prefix` option             | Namespace all routes in a plugin                   |
| `dependencies`              | Declare plugin load order requirements             |
| `@fastify/autoload`         | Automatically load plugins from directories        |

---

## Summary

1. **Everything is a plugin** — Embrace this mental model
2. **Encapsulation by default** — Plugins isolate their additions
3. **Use `fp()` sparingly** — Only for shared infrastructure
4. **Async plugins** — Always use the async pattern
5. **Decorators need shapes** — Provide initial values for performance
6. **Test in isolation** — Plugins are independently testable
7. **Organize by feature** — Group related routes/logic in encapsulated plugins
