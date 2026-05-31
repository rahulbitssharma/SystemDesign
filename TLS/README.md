# TLS (Transport Layer Security)

A practical, system-design focused guide to TLS internals: handshake flow, certificate validation, record format, performance, and operational troubleshooting.

## Table of Contents

1. **[Introduction](01-introduction.md)** — Why TLS exists, threat model, and where it sits in the network stack.
2. **[TLS 1.3 Handshake Transcript](02-tls13-handshake-transcript.md)** — Message-by-message flow and security meaning.
3. **[TLS Record Layer and Packet Mapping](03-record-layer-and-packet-mapping.md)** — How plaintext becomes TLS records over TCP/IP.
4. **[OpenSSL in Modern Applications](04-openssl-in-modern-applications.md)** — How applications use OpenSSL in practice, from sockets to encrypted I/O.
5. **[OpenSSL vs Other TLS Libraries](05-openssl-vs-other-tls-libraries.md)** — When teams choose OpenSSL, BoringSSL, NSS, or rustls and why.
6. **[TLS Troubleshooting and Runbook](06-tls-troubleshooting-and-runbook.md)** — Practical failure modes, debugging flow, and operational playbooks.

---

## Quick Start

**New to TLS?** Start with [Introduction](01-introduction.md).

**Need handshake clarity?** Go to [TLS 1.3 Handshake Transcript](02-tls13-handshake-transcript.md).

**Debugging packet traces?** Read [TLS Record Layer and Packet Mapping](03-record-layer-and-packet-mapping.md).

**Need implementation-level code paths?** Go to [OpenSSL in Modern Applications](04-openssl-in-modern-applications.md).

**Need library trade-offs?** Read [OpenSSL vs Other TLS Libraries](05-openssl-vs-other-tls-libraries.md).

**Need incident/debug guidance?** Read [TLS Troubleshooting and Runbook](06-tls-troubleshooting-and-runbook.md).
