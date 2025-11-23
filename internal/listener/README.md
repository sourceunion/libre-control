# Shared Listener Logic

This package contains the reusable core logic for network communication, preventing code duplication across different Listener implementations (HTTP, DNS, SMB).

> [!NOTE]
> **DRY (Don't Repeat Yourself)**. If logic regarding how to validate an Agent ID is used by both HTTP and DNS listeners, it belongs here, not in the `cmd` folders.

## Purpose

While `cmd/listener_http` and `cmd/listener_dns` contain the specific transport bindings, this internal package handles the **common denominator** tasks:

- **Request Normalization**: Converting diverse inputs (an HTTP POST body or a DNS TXT record) into a standardized internal struct.
- **Traffic Filtering**: Logic to identify and drop "orphaned" packets or obvious scans before they hit the Broker.
- **Concurrency Control**: Worker pools for handling high-volume ingress traffic.