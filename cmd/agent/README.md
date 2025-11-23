# The Remote Agent (Implant)

This directory contains the source code for the **Implant**, the artifact deployed onto target systems.

## Operational Role

The Agent is the execution engine of the LibreControl framework. It operates in hostile environments and is responsible for:

1. **Beaconing**: Periodically checking in with the C2 infrastructure.
2. **Task Execution**: Running system commands, injecting code, or manipulating files.
3. **Reporting**: Encrypting and returning results to the Listener.

## Critical Build Constraint

> [!TIP]
> Do not run `go run main.go` here directly.

This code is designed to be compiled by the **Factory Service** (`cmd/factory`). The Factory injects critical configuration data (C2 IP, Public Key, Jitter) at compile-time using linker flags (`-ldflags`). Running it directly without these flags will result in a binary with no configuration, which will immediately exit or fail to connect.

## Architecture Boundaries

- **Zero Shared State**: This binary **must not** import `internal/database` or `internal/broker`. It has no concept of the backend infrastructure.
- **Standard Library Only**: To keep the binary size small (~3MB) and avoid dependency hell on target machines, external dependencies are strictly minimized.