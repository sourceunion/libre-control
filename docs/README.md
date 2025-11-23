# Operational Documentation & Governance

This directory serves as the central repository for the non-technical frameworks that govern the use of **LibreControl**. While the root `README.md` and `ARCHITECTURE.md` focus on software engineering, the documents herein focus on **Operational Security (OpSec)**, legal compliance, and risk management.

## Document Index

### [Risk Assessment & Security Protocols](RISK_ASSESSMENT.md)

**Mandatory Reading for All Operators.** This document outlines the critical security postures required to run this framework safely. It covers:

- **Infrastructure Hardening**: Network segregation rules to prevent server compromise.
- **Cryptographic Hygiene**: Key lifecycle management and risks of attribution.
- **Legal Boundaries**: The "Rules of Engagement" and audit requirements for authorized simulations.
- **Incident Response**: Protocols for "burning" the infrastructure in case of detection or compromise.

### [Operator Field Manual (TTPs)](OPERATOR_MANUAL.md)

**The Standard Operating Procedure (SOP).** A comprehensive guide bridging technical architecture and operational execution. It details:

- **Infrastructure Initialization**: Boot sequences and health checks.
- **Weaponization**: Payload generation parameters and artifact handling.
- **Tradecraft**: Advanced techniques for persistence, lateral movement, and data exfiltration.
- **Forensics Evasion**: Anti-forensics, timestomping, and traffic masquerading.