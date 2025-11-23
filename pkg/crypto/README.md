# Cryptographic Primitives

> [!CAUTION]
> Do not roll your own crypto. This package wraps standard library implementations (`golang.org/x/crypto`) to enforce specific parameters. Do not modify the curve parameters or nonce generation logic without rigorous review.

This package implements the secure channel protocols for LibreControl. It provides the mathematical foundation for confidentiality and integrity.

## Standards

- **Key Exchange**: Elliptic Curve Diffie-Hellman (ECDH) over **Curve25519 (X25519)**.
- **Key Derivation**: HMAC-based Extract-and-Expand Key Derivation Function (**HKDF**) using SHA-256.
- **Symmetric Encryption**: **ChaCha20-Poly1305** (AEAD).
    
## Implementation Details

### The Handshake

1. **Agent** generates an ephemeral keypair.
2. **Agent** computes shared secret using Server's Static Public Key.
3. **Agent** derives Session Keys ($K_{tx}, K_{rx}$) via HKDF.
4. **Server** receives ephemeral public key and performs the inverse operation.

### Security Guarantee

This implementation aims for **Perfect Forward Secrecy (PFS)** per session. The compromise of a session key exposes only that specific session's traffic, not historical data.