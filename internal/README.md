# Internal Application Logic

This directory contains the private business logic of the LibreControl framework.

## The "Internal" Rule

In Go, packages within an `internal/` directory **cannot be imported** by code outside the parent tree rooted at `internal/`. This is an architectural enforcement mechanism to ensure that:

1. The **Agent** (which is an external binary) never accidentally imports server-side logic (like database drivers).
2. The API surface remains clean.

## Modules

- **`server/`**: Contains the core management logic (Task queueing, Operator authentication, Session management).
- **`database/`**: Handles SQL queries, migrations, and persistence (PostgreSQL).
- **`broker/`**: Adapters for the message bus (Redis/RabbitMQ), abstracting the Pub/Sub complexity.
- **`listener/`**: Common logic shared _only_ between listener implementations (e.g., generic request parsing).