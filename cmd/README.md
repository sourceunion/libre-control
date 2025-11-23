# Application Entry Points

This directory contains the "Main" packages for the executable artifacts of the LibreControl framework.

## Philosophy

Per Standard Go Layout, the code within these directories is minimal. It is responsible strictly for **bootstrapping**:

1. Parsing command-line arguments/flags.
2. Loading configuration.
3. Initializing dependencies (Loggers, Database connections).
4. Invoking the business logic residing in `internal/`.

## Artifacts

- **`teamserver/`**: The central orchestration brain. Connects to the Database and Broker.
- **`listener_http/`**: A specialized listener for HTTP/S traffic. Stateless.
- **`listener_dns/`**: A specialized listener for DNS tunneling. Stateless.
- **`factory/`**: The payload generation service.
- **`agent/`**: The source code for the remote implant. _Note: This directory is read by the Factory service to generate binaries._