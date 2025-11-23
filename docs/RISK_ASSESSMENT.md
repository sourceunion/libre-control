# Security & Operational Risk Assessment

This document establishes the mandatory security protocols for the deployment and operation of **LibreControl**. Given the nature of Command and Control (C2) frameworks, the risk profile is twofold:

1. **Inbound Risk**: The C2 infrastructure itself is a high-value target for rival threat actors or counter-intelligence operations. Vulnerabilities here can lead to the operator's compromise.
2. **Outbound Risk**: Operational mistakes can lead to legal liability, attribution, or unintended damage to target infrastructure.

> [!WARNING]
> This framework is provided "as-is" for educational purposes. The safeguards described herein are the sole responsibility of the operator.

## Infrastructure Security (AppSec & NetSec)

The architecture relies on strict isolation and controlled trust boundaries. Any deviation from the prescribed `docker‑compose` topology — especially exposing internal containers or flattening network layers — creates severe security risks, including lateral movement, unauthorized access, and full compromise of the Team Server.

### Network Segregation

The **Team Server** and **Database** operate inside a protected, non-routable trust zone. These components **MUST NOT** be directly reachable from the public internet under any circumstances, as they contain the system’s authoritative state, cryptographic material, and operational history.

- **Requirement**: The `docker-compose` network `c2_grid` must be configured with `internal: true` and protected behind strict firewall rules. This ensures that only explicitly allowed services — typically the Listener and Message Broker — can communicate with the Team Server and Database.
- **Redirectors**: A Listener should never be exposed directly to targets or external networks. Instead, use a dedicated **Redirector** (e.g., Nginx, Apache, HAProxy) hosted on a separate VPS or cloud node. The Redirector terminates public traffic, applies filtering and rate-limiting, and forwards only sanitized requests to the Listener. If the Redirector is compromised, misconfigured, or "burned", the Listener and core services remain hidden, preventing adversaries from attacking internal infrastructure.
- **Firewall & ACL Enforcement**: Apply explicit ingress/egress rules on every layer — Redirector, Listener, Team Server, and Database — to restrict traffic to required ports and known peers. Deny-all defaults and allow-list policies reduce the attack surface and prevent accidental exposure.
- **Container Boundary Protection**: Ensure each service runs in an isolated container or namespace with minimal privileges, preventing cross-container compromise or escalation if one component is attacked.

### Secrets Management

The framework utilizes cryptographic secrets for authentication and encryption.

- **JWT Secret**: The environment variable `JWT_SECRET` in `docker-compose.yml` is **strictly enforced**. The application utilizes a "Fail-Secure" initialization. If `JWT_SECRET` is unset, empty, or matches the default placeholder, the Team Server **will refuse to start** (Fatal Error).
- **Database Credentials**: The DBMS password must be high-entropy (minimum 20 characters), unique, and rotated regularly. Hard-coded or weak passwords are strictly prohibited, and access should be limited via role-based permissions.
- **Secret Rotation & Expiration**: All long-lived secrets should be periodically rotated and invalidated to reduce exposure if compromised. Automated rotation and versioning are recommended for operational reliability.
- **In-Memory Handling**: Secrets are loaded into memory only as needed and never written to disk or logs. Any temporary buffers are cleared immediately after use to minimize exposure to memory dumps or process inspection.

### Input Sanitization & RCE

C2 servers are historically vulnerable to **"Hot Patching" attacks**, where a malicious or compromised Agent sends malformed data in an attempt to trigger remote code execution or corrupt server memory. The Listener component acts as the first defensive barrier, enforcing strict validation, isolation, and predictable parsing behavior to eliminate this class of attacks.

#### Defensive Parsing & Boundary Enforcement

- **Max Size Enforcement**: All incoming beacons are hard‑capped (e.g., 5MB) *before* any parsing or allocation occurs. This prevents memory‑exhaustion DoS attacks and ensures that oversized or intentionally bloated payloads are discarded without impacting server stability.
- **Strict Schema Validation**: The parser uses strong typing (e.g., Protobuf, JSON with strict mode) and rejects unknown, optional, or out‑of‑spec fields, blocking malicious attempts to bypass logic via deserialization gadgets or unexpected control-flow manipulation.
- **Canonicalization & Encoding Normalization**: Input fields are normalized into a canonical representation prior to processing, preventing attacks that rely on multi-encoding tricks (e.g., Unicode smuggling, mixed UTF‑8/UTF‑16 sequences, or ambiguous escape handling).
- **Sandboxed Parsing Environment**: Complex or deeply nested inputs are parsed inside a constrained execution context to reduce blast radius, ensuring that even parser-level bugs cannot escalate into RCE.
- **Rate & Anomaly Detection**: Incoming beacon patterns are rate-limited and checked for structural anomalies — such as repeated parse failures or malformed sequences — to detect probing activity or iterative exploit attempts early.

## Cryptographic Hygiene

The security of the communications depends entirely on the secrecy of the cryptographic keys.

### Key Lifecycle & Forward Secrecy

Ensures that cryptographic operations maintain strong confidentiality, integrity, and resilience against key compromise. Key material is strictly compartmentalized between authentication and encryption purposes, with ephemeral keys protecting operational traffic.

- **Authentication vs. Encryption**: The static private key ($S_{priv}$) is used **exclusively for authentication** (signing messages, verifying identity). It is never used to directly encrypt session traffic, reducing exposure in case of compromise.
- **Forward Secrecy (FS)**: Confidentiality is enforced via **ECDHE (Elliptic Curve Diffie-Hellman Ephemeral)**. A unique, ephemeral session key is generated for each interaction and discarded immediately after use, ensuring that each session remains isolated.
- **Key Expiration & Rotation**: Ephemeral keys are short-lived and automatically invalidated after use. Static authentication keys should be rotated periodically according to operational security policy.
- **Impact of Compromise**: Even if the server is seized and $S_{priv}$ is recovered, prior traffic captures remain indecipherable. Forward secrecy ensures that historical sessions cannot be decrypted retroactively.
- **In-Memory Key Handling**: Ephemeral session keys exist only in volatile memory during the session and are zeroed out immediately upon expiration to prevent leakage via memory inspection or crash dumps.

### Artifact Signatures

Agents and binaries produced by the framework have identifiable characteristics that must be considered when planning deployment or operational security. Proper awareness helps avoid unintentional exposure despite encrypted communications.

- **Binary Analysis**: Agents generated by `cmd/factory` are un-obfuscated Go binaries. They retain readable strings, function names, and static signatures, which are easily detectable by reverse engineering or automated scanning tools.
- **EDR & AV Risk**: Modern Endpoint Detection & Response (EDR) and antivirus solutions will flag these binaries immediately. Encryption of network traffic does **not** obfuscate the binary itself; on-disk artifacts remain identifiable.
- **Mitigation Considerations**: Any operational deployment must account for on-disk signatures. Techniques such as controlled execution environments, ephemeral containers, or isolated test networks reduce exposure, though detection cannot be entirely eliminated.
- **Operational Awareness**: Teams should maintain full awareness that artifact signatures may betray activity patterns. Audit trails, logging, and container isolation should be used to minimize footprint and prevent accidental compromise of hosts.

## Operational Security (OpSec)

Maintaining strong operational security is critical to prevent attribution, exposure, or compromise of the management infrastructure. All actions should assume that network traffic and artifacts can be monitored or analyzed by adversaries.

### Attribution & Origin

- **Source IP Protection**: Never operate the Console (`cmd/teamserver` API) from a home or corporate IP address. Always use trusted anonymization layers such as VPNs, Tor, or dedicated jump hosts to isolate operational origin and prevent direct attribution.
- **Traffic Fingerprinting**: Default HTTP headers from Listeners reveal the server as LibreControl. You **MUST** configure Malleable Profiles to mimic legitimate server signatures (e.g., Apache/2.4.50) and standardize response behaviors to blend into normal traffic patterns. This reduces the risk of automated detection, censorship, or targeted blocking by security teams.
- **Operational Isolation**: Management systems should reside on air-gapped or logically separated networks wherever possible. Avoid combining administrative access with external-facing infrastructure to reduce the attack surface.
- **Audit Minimization**: Limit verbose logging or error messages that could leak system information. Only store necessary telemetry, and secure audit logs to prevent leakage of operational patterns.
- **Compartmentalization of Tasks**: Use role separation, temporary credentials, and ephemeral environments for sensitive operations to minimize the impact of accidental exposure or compromise.

### Deconfliction & Tamper‑Evident Logs

Robust auditing and traceability are essential for maintaining accountability, supporting deconfliction between teams, and providing verifiable evidence during authorized security assessments. The logging system is designed to prevent silent manipulation and ensure that all operational actions remain provably authentic.

- **Traceability**: The `internal/database` subsystem maintains a **Cryptographically Chained Log** (HMAC Ledger). Each entry is linked to the previous one, creating a verifiable, sequential chain of events that supports accurate reconstruction of activity during incident reviews or engagement debriefs.
- **Integrity Enforcement**: Every log entry includes a hash of the previous entry, forming an immutable ledger. Any attempt to delete, insert, or modify a record — even by a privileged database administrator (root) — breaks the chain and becomes immediately detectable. This ensures reliable, tamper‑evident auditing.
- **Operational Protocol**: During authorized Red Team or security engagements, you may be contractually required to provide these logs to the client for verification, deconfliction, or legal validation of your actions. **DO NOT DELETE LOGS**; they serve as your cryptographically verifiable activity record and are essential for transparency, compliance, and liability protection.
- **Retention & Safeguarding**: Logs should be archived securely, encrypted at rest, and backed up to a trusted storage location. Retention policies must comply with organizational and contractual requirements to guarantee availability during post-engagement reporting or dispute resolution.

## Legal Boundaries & Ethics

Strict adherence to legal and ethical standards is mandatory for all operations. Violating these principles may result in civil or criminal liability. All team members must ensure that every action is authorized, scoped, and compliant with applicable laws.

1. **Authorization:** You must possess **written, signed authorization** (Scope of Work or equivalent legal document) from the owner of the target infrastructure before deploying any Agent. Unauthorized operations are strictly prohibited.
2. **Scope Creep:** Operations must remain strictly within the agreed-upon scope. Do **not** pivot to other networks, hosts, or systems even if they are accessible from the target environment. Any deviation must be explicitly approved.
3. **Data Privacy & Handling:** Avoid capturing or exfiltrating PII (Personally Identifiable Information) unless explicitly required for the engagement. Any inadvertently collected PII — such as credentials, screenshots, or documents — must be securely deleted immediately to prevent misuse or legal liability.
4. **Accountability & Compliance:** Maintain detailed, verifiable records of all actions. These logs, coupled with tamper-evident audit trails, serve as a legal shield and evidence of ethical operation.

> [!CAUTION]
> Violation of these parameters may constitute a criminal offense, and could lead to severe legal consequences.

## Incident Response (Self-Compromise)

In the event that the C2 infrastructure is suspected to have been compromised by a third party, immediate and decisive action is required to contain exposure, prevent escalation, and restore operational security.

1. **Kill Switch:** Execute `docker-compose down -v` immediately to sever all active connections and terminate containers, preventing further interaction with deployed Agents.
2. **Burn Compromised Assets:** Assume all deployed Agents and infrastructure are under adversary control. Perform manual remediation on target hosts and remove any persistent artifacts to prevent further misuse.
3. **Key Rotation & Rebuild:** Generate new cryptographic keys and rebuild the infrastructure from a trusted state. Avoid reusing compromised credentials or configuration files. All ephemeral keys and session material must be invalidated.
4. **Post-Incident Verification:** Audit all logs, system states, and communications to ensure that no residual compromise persists. Verify that restored operations meet security and operational standards before resuming normal activity.