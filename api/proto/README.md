# Protocol Buffer Definitions (.proto)

This directory contains the raw Protocol Buffer schema files. These files act as the **Interface Definition Language (IDL)** for the LibreControl framework.

## Purpose

In a C2 architecture where the **Agent** (Implant) and the **Server** (Core) are physically and logically separated, strict data typing is essential to prevent parsing errors and buffer vulnerabilities.

These `.proto` files are **not** used directly by the Go compiler. Instead, they are used to **generate** the Go code (structs and serialization methods) that resides in `pkg/protocol`.

## Workflow

1. **Edit**: Developers modify the schema here (e.g., adding a new field to `Task`).
2. **Generate**: Run the `protoc` compiler to update the Go bindings.
3. **Commit**: Both the `.proto` files and the generated `.pb.go` files (in `pkg/`) must be committed.

## Best Practices

- **Backward Compatibility**: Never change the field ID (e.g., `int32 id = 1;`) of an existing field. Doing so will break communication with older Agents already deployed in the field.
- **Field Deprecation**: If a field is no longer needed, mark it as `reserved` rather than deleting it.