# Shared Public Libraries

This directory contains code designed to be shared across different binaries within the monorepo, specifically between the **Server Infrastructure** and the **Remote Agent**.

## Usage Policy

Unlike `internal/`, packages here are public. They must be stateless and free of heavy external dependencies (like SQL drivers) to ensure the Agent binary remains lightweight.

## Key Packages

- **`crypto/`**: The cryptographic core. Implements the X25519 Key Exchange, HKDF derivation, and ChaCha20-Poly1305 wrappers. **Critical Security Component.**
- **`protocol/`**: Go structs and definitions shared by the C2 and the Agent to ensure they speak the same language (e.g., JSON/Binary formats).
- **`utils/`**: Generic helpers (Logging interfaces, Random string generation, UUID handling).
- **`jitter/`**: Mathematical implementations for sleep obfuscation and beacon timing.