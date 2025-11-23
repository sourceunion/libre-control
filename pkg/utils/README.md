# General Utilities

This package contains agnostic helper functions that do not fit into specific domain logic.

## Scope

- **Logging**: Standardized interfaces for structured logging (Info/Warn/Error).
- **Identity**: Functions to generate UUIDs or high-entropy random strings for Agent IDs.
- **Sanitization**: Helpers to clean strings or file paths.

## Constraint

This package must have **zero dependencies** on the `internal/` logic. It must remain purely functional and stateless to ensure it can be safely imported by the lightweight Agent binary without bloating the file size.