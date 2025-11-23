# API & Protocol Definitions

This directory contains the **canonical data contracts** that govern the communication between the distributed microservices of LibreControl.

## Purpose

In a distributed system, independent services (Core, Listeners, Factory) must agree on the shape of data without sharing implementation details. This directory serves as the single source of truth for those definitions.

## Contents

- **`proto/`**: Protocol Buffer (`.proto`) definitions. These are language-agnostic schemas used to generate the Go structs for gRPC or serialization logic.

## Architecture Note

Modifications to the message structures should originate here. Do not manually modify generated Go files in the `pkg/` directory without updating the definitions here first.