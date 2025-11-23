# Team Server Business Logic

This package serves as the "Brain" of the LibreControl framework. It implements the specific rules of engagement and orchestration.

## Core Components

- **Session Manager**: Tracks active agents, last seen times, and encryption keys.
- **Task Scheduler**: Logic to queue commands for agents that are currently offline (asynchronous tasking).
- **Operator API**: Handlers for the Frontend/Console requests (authentication, command issuing).

## Dependency Flow

This package coordinates the `internal/database` and `internal/broker` packages. It receives an intent from an operator, persists it to the DB, and pushes it to the Broker.