# Protocol Definitions & Data Structures

This package defines the **Shared Contract** between the Agent and the Server.

## Importance

Since the Agent and Server are separate binaries (often running on different operating systems), they must agree exactly on the binary layout of data. This package contains the Go `structs` that map to the JSON or Binary payloads exchanged over the wire.

## Contents

- **`Beacon`**: Structure for the initial handshake (metadata, user ID, PID).
- **`Task`**: Structure for a command sent to the agent (TaskID, OpCode, Arguments).
- **`Response`**: Structure for the result returned by the agent (TaskID, Output, ErrorCode).

## Usage

Both `cmd/agent` and `cmd/teamserver` import this package. Changing a struct here affects both sides of the communication.