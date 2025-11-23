# Team Server Core

This directory contains the **Orchestrator**, the central brain of the LibreControl framework.

## Operational Role

The Team Server is the only component that communicates with the **Database**. Its responsibilities include:

1. **State Management**: Maintaining the list of active agents and their metadata.
2. **Cryptography**: Holding the private keys required to decrypt agent traffic.
3. **API Hosting**: Exposing the REST/GraphQL API used by the Operator's Console (CLI/GUI).

## Dependency Flow

- **Inbound**: Consumes messages from the `broker`.
- **Outbound**: Persists data to `database` and publishes tasks to `broker`.

## Access Control

This binary should **never** be exposed directly to the internet. It should reside within a protected management VLAN (or internal Docker network), accessible only via the Operator Console or a VPN.