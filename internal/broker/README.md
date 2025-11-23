# Message Broker Adapter

> [!NOTE]
> While the default implementation may use **Redis**, the code in the rest of the application (`cmd/teamserver`, `cmd/listener`) must never import `github.com/redis/go-redis` directly. They must interact only with this package's interface. This allows swapping Redis for RabbitMQ or NATS in the future with zero refactoring of the core logic.

This package provides the abstraction layer for the asynchronous event bus that decouples the Team Server from the Listeners.

## Architectural Role

The **Broker** acts as the central nervous system. It implements the **Publisher/Subscriber** and **Queue** patterns. Crucially, this package creates an **Interface Adapter** (Port & Adapter Architecture).

## Interfaces

This package must expose generic interfaces.