# The Payload Factory

This directory contains the microservice responsible for the **Just-in-Time (JIT) Compilation** of agents.

## Operational Role

The Factory listens to the Message Broker for build requests (`build_queue`). When a request arrives, it:

1. **Configures**: Reads the requested parameters (Target OS, C2 Protocol, Encryption Keys).
2. **Compiles**: Invokes the Go Toolchain to cross-compile the `cmd/agent` source code.
3. **Obfuscates**: Applies AST manipulation to alter the binary signature.
4. **Publishes**: Returns the finished binary artifact via the Broker.

## Environment Requirements

Unlike other microservices in this project which run on `alpine:latest`, this service requires a container image with the full **Go Toolchain** installed. It must also have filesystem access to the `cmd/agent` and `pkg/` directories to use them as templates.