# Persistence & Data Access Layer

This package encapsulates all interactions with the relational database (PostgreSQL).

## Responsibilities

1. **Schema Migration**: Defining the structure of tables (`agents`, `tasks`, `listeners`) and handling versioned updates.
2. **Data Access Objects (DAO)**: Providing strongly-typed methods to Create, Read, Update, and Delete (CRUD) operational data.
3. **Audit Logging**: Ensuring that every state change (e.g., "User X executed Command Y") is written to an immutable log for post-mortem analysis.

## Constraints

- **Isolation**: Only the **Team Server** (`cmd/teamserver`) is permitted to import this package.
- **No Direct SQL**: Listeners and Agents must **never** access the database directly. They must interact with the system via the _Message Broker_ or _API_.