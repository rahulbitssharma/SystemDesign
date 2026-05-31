# TLS — Introduction

TLS secures application traffic over insecure networks by providing three core guarantees:

- **Confidentiality**: encrypts data so intermediaries cannot read plaintext.
- **Integrity**: detects tampering in transit.
- **Authentication**: validates server identity (and optionally client identity).

## Where TLS Sits in the Stack

For HTTPS over TCP, the stack is:

`HTTP` -> `TLS` -> `TCP` -> `IP` -> `Link`

For HTTP/3, TLS runs with QUIC over UDP rather than directly over TCP.

## Why TLS Matters in System Design

- Protects user and service data on untrusted networks.
- Enables secure service-to-service communication.
- Establishes identity boundaries using certificates and trust roots.
- Affects latency due to handshake and key exchange.

## Common Deployment Realities

- TLS can terminate at edge load balancers or reverse proxies.
- Back-end hops may be re-encrypted (TLS) or plaintext depending on policy.
- Certificate lifecycle and rotation are operationally critical.

---

**Navigation:**
- [Back to Index](README.md)
- [Next: TLS 1.3 Handshake Transcript ->](02-tls13-handshake-transcript.md)
