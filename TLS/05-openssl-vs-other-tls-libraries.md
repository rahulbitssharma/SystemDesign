# TLS — OpenSSL vs Other TLS Libraries

Modern applications rarely choose a TLS library by accident. The library choice affects:

- API style and integration effort
- security hardening defaults
- platform portability
- protocol feature support
- operational debugging behavior
- performance and footprint

This chapter compares the libraries most commonly encountered in practice:

- OpenSSL
- BoringSSL
- NSS
- rustls

## Comparison Dimensions

The most practical comparison dimensions are:

1. Who maintains it?
2. What applications commonly use it?
3. Is API/ABI stability a goal?
4. Is it general-purpose or tightly ecosystem-specific?
5. What language/runtime integration model is typical?
6. What is the operational/debugging experience like?

## OpenSSL

### What It Is

OpenSSL is a broad-purpose TLS and crypto library with a long history and wide deployment in servers, tools, middleware, and embedded software.

### Typical Usage

- web servers
- reverse proxies
- CLIs and network tools
- SDKs and legacy enterprise software
- language runtimes and wrappers

### Strengths

- very widely available
- rich API surface
- broad protocol and X.509 support
- large ecosystem and tooling familiarity

### Trade-Offs

- large API surface can be complex
- historical compatibility baggage
- misuse-prone if configuration is weak

### C-Style Pseudo-Code: Typical OpenSSL Integration Shape

```c
SSL_CTX *ctx = SSL_CTX_new(TLS_client_method());
SSL *ssl = SSL_new(ctx);
SSL_set_fd(ssl, fd);
SSL_connect(ssl);
SSL_write(ssl, req, req_len);
SSL_read(ssl, resp, sizeof(resp));
```

## BoringSSL

### What It Is

BoringSSL is a fork of OpenSSL maintained primarily for Google's needs. It is used heavily in Chromium and related ecosystems.

### Typical Usage

- Chromium/Chrome family
- Android components
- applications willing to vendor library code

### Strengths

- streamlined for selected consumers
- strong fit for projects that vendor dependencies
- aggressive cleanup and simplification relative to compatibility-focused OpenSSL usage

### Trade-Offs

- not intended as a general stable drop-in library for everyone
- API/ABI stability is not a design goal in the same way traditional system libraries are

### Practical Meaning

Teams usually do not pick BoringSSL as a system dependency unless they are comfortable vendoring and tracking upstream changes closely.

## NSS

### What It Is

NSS (Network Security Services) is Mozilla's security library stack, used heavily in Firefox and related Mozilla infrastructure.

### Typical Usage

- Firefox / Gecko ecosystem
- applications tied to Mozilla security stack

### Strengths

- mature certificate and PKI support
- strong integration in Mozilla ecosystem

### Trade-Offs

- less commonly used as the default choice for generic non-Mozilla application development
- API style differs from OpenSSL-family expectations

## rustls

### What It Is

rustls is a modern TLS library written in Rust, with a design emphasis on memory safety and simpler safe-by-default integration patterns.

### Typical Usage

- Rust services and clients
- modern security-conscious greenfield projects

### Strengths

- memory-safety advantages from Rust
- smaller/simpler integration story for many Rust apps
- good fit for modern application codebases that do not require full OpenSSL-style API breadth

### Trade-Offs

- not a drop-in answer for C/C++ ecosystems
- some operational environments still expect OpenSSL-compatible behavior or tooling

## Practical Decision Matrix

| Library | Common Fit | Why Teams Pick It |
|---|---|---|
| OpenSSL | General-purpose C/C++ servers and clients | Ubiquity, tooling familiarity, broad compatibility |
| BoringSSL | Vendored app/browser stacks | Tight control, ecosystem alignment, curated API |
| NSS | Mozilla-centric systems | Existing ecosystem integration |
| rustls | Rust-native services and clients | Safer implementation model, modern integration style |

## Deep Dive: One Practical Choice Example

Assume a team is building a reverse proxy in C/C++ that must:

- run on Linux across many environments
- support mTLS
- integrate with existing PEM certificate/key workflows
- expose TLS version/cipher telemetry
- work with existing operational knowledge and debug tooling

A common decision is OpenSSL because:

- deployment environments already package it
- engineers already know its tooling and errors
- PEM/X.509 workflows are straightforward
- ecosystem examples are abundant

### C-Style Pseudo-Code: Why OpenSSL Is Operationally Convenient

```c
SSL_CTX *ctx = SSL_CTX_new(TLS_server_method());
SSL_CTX_use_certificate_chain_file(ctx, "/etc/tls/server-chain.pem");
SSL_CTX_use_PrivateKey_file(ctx, "/etc/tls/server-key.pem", SSL_FILETYPE_PEM);
SSL_CTX_set_verify(ctx, SSL_VERIFY_PEER | SSL_VERIFY_FAIL_IF_NO_PEER_CERT, NULL);
SSL_CTX_load_verify_locations(ctx, "/etc/tls/ca-bundle.pem", NULL);
```

This aligns well with standard file-based cert deployment and established operations playbooks.

## Another Practical Choice Example: Browser Stack

Assume a browser vendor wants:

- very tight control over TLS internals
- a vendored dependency model
- aggressive refactoring freedom
- integration with browser-specific networking and sandboxing

A library like BoringSSL is a more natural fit.

### Conceptual Integration Shape

```c
browser_tls_config_t cfg = browser_build_tls_config();
boringssl_session_t *sess = browser_tls_session_new(cfg);
browser_tls_handshake(sess, fd);
```

The point is not the API spelling but the ownership model: the application owns the stack deeply.

## Operational Considerations Beyond Code

When choosing a TLS library, teams should also evaluate:

1. How are CVEs tracked and patched?
2. Can the dependency be updated safely in production?
3. What metrics, logs, and debug tools already exist internally?
4. Does the platform already standardize on one library?
5. Is FIPS or platform certification required?

## Recommendation Heuristic

A practical rule of thumb:

1. Choose OpenSSL for broad compatibility and conventional C/C++ infrastructure.
2. Choose BoringSSL only if vendoring and ecosystem alignment are intentional.
3. Choose NSS mainly when already in Mozilla-oriented ecosystems.
4. Choose rustls for Rust-first systems where safety and modern integration dominate.

---

**Navigation:**
- [<- Previous: OpenSSL in Modern Applications](04-openssl-in-modern-applications.md)
- [Back to Index](README.md)
- [Next: TLS Troubleshooting and Runbook ->](06-tls-troubleshooting-and-runbook.md)
