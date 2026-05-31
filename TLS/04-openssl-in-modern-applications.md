# TLS — OpenSSL in Modern Applications

OpenSSL is one of the most widely used TLS and cryptography libraries in modern systems. It provides:

- TLS client and server protocol implementation
- Certificate parsing and validation primitives
- Symmetric/asymmetric crypto primitives
- X.509 and PKI utilities
- BIO abstractions for socket and memory I/O

This chapter explains what OpenSSL does in a real application, how applications call into it, and what the code path looks like from socket creation to encrypted reads and writes.

## Where OpenSSL Fits

In a typical application stack using OpenSSL:

`Application protocol logic` -> `OpenSSL TLS state machine` -> `OS socket API` -> `TCP/IP stack`

OpenSSL does not replace the OS network stack. Instead:

- OS handles socket creation, TCP connection state, packet transmit/receive, buffering.
- OpenSSL drives TLS handshake, certificate validation policy hooks, record encryption/decryption.
- Application code handles plaintext protocol logic such as HTTP parsing.

## What Modern Applications Use OpenSSL For

Common usage patterns:

1. HTTPS clients
- browsers, CLIs, service clients, SDKs

2. HTTPS/TLS servers
- web servers, API gateways, service meshes, internal control planes

3. Mutual TLS service-to-service communication
- internal microservices, control planes, mesh sidecars

4. TLS termination at proxies/load balancers
- reverse proxies, ingress controllers, edge gateways

## Practical Example: HTTPS Client in C Using OpenSSL

We will use one concrete application model throughout this chapter:

- App opens TCP socket to `www.example.com:443`
- App uses OpenSSL to perform TLS handshake
- App sends plaintext HTTP request through OpenSSL
- OpenSSL encrypts request into TLS records and writes to socket
- OpenSSL reads encrypted response bytes from socket and returns plaintext HTTP response bytes to app

## Phase 1: Initialize OpenSSL Library State

In modern OpenSSL versions, initialization is lighter than in older releases, but applications still create reusable SSL contexts.

### What SSL_CTX Represents

`SSL_CTX` is the process- or configuration-level TLS context. It usually contains:

- protocol method selection
- trust store settings
- verification policy
- default cipher configuration
- certificate/key material for server or client auth

### C-Style Pseudo-Code: Create TLS Client Context

```c
SSL_CTX *ctx = SSL_CTX_new(TLS_client_method());
if (ctx == NULL) {
	/* handle error */
}

SSL_CTX_set_verify(ctx, SSL_VERIFY_PEER, NULL);
SSL_CTX_set_default_verify_paths(ctx);
SSL_CTX_set_min_proto_version(ctx, TLS1_2_VERSION);
+```

Why this matters:

- `TLS_client_method()` creates a client-capable context.
- verification must be explicitly enabled for real security.
- trust paths/root stores determine which certificate chains are accepted.

## Phase 2: Open TCP Socket

OpenSSL does not magically create IP packets by itself. The application still uses normal OS networking calls.

### C-Style Pseudo-Code: Resolve and Connect

```c
int fd = socket(AF_INET, SOCK_STREAM, 0);
if (fd < 0) {
	/* handle error */
}

connect(fd, (struct sockaddr *)&server_addr, sizeof(server_addr));
+```

In a real client, this often includes:

- DNS resolution using `getaddrinfo`
- trying IPv6/IPv4 candidates
- connect timeout handling
- proxy support if configured

### C-Style Pseudo-Code: getaddrinfo + connect Loop

```c
struct addrinfo hints = {0};
struct addrinfo *res = NULL, *rp = NULL;
hints.ai_family = AF_UNSPEC;
hints.ai_socktype = SOCK_STREAM;

getaddrinfo("www.example.com", "443", &hints, &res);

int fd = -1;
for (rp = res; rp != NULL; rp = rp->ai_next) {
	fd = socket(rp->ai_family, rp->ai_socktype, rp->ai_protocol);
	if (fd < 0) continue;
	if (connect(fd, rp->ai_addr, rp->ai_addrlen) == 0) {
		break;
	}
	close(fd);
	fd = -1;
}
freeaddrinfo(res);
+```

## Phase 3: Bind OpenSSL Session to Socket

`SSL` is the connection-level TLS object, distinct from `SSL_CTX`.

### What SSL Represents

`SSL` holds per-connection state:

- handshake state machine
- negotiated cipher suite
- transcript state
- traffic keys
- peer certificate chain
- read/write BIO bindings

### C-Style Pseudo-Code: Create SSL and Attach FD

```c
SSL *ssl = SSL_new(ctx);
if (ssl == NULL) {
	/* handle error */
}

SSL_set_fd(ssl, fd);
SSL_set_tlsext_host_name(ssl, "www.example.com");
SSL_set1_host(ssl, "www.example.com");
+```

Why both hostname calls matter:

- `SSL_set_tlsext_host_name` sets SNI for server-side certificate/site selection.
- `SSL_set1_host` sets expected hostname for verification.

## Phase 4: Drive the TLS Handshake

### What `SSL_connect()` Does Internally

`SSL_connect()` drives the TLS client state machine. Internally it may:

1. build ClientHello
2. write handshake bytes to socket
3. read ServerHello and subsequent messages
4. validate certificate chain and hostname
5. derive traffic secrets
6. complete Finished exchange

### C-Style Pseudo-Code: Blocking Handshake

```c
int ret = SSL_connect(ssl);
if (ret <= 0) {
	int err = SSL_get_error(ssl, ret);
	/* handle SSL_ERROR_SSL, SSL_ERROR_SYSCALL, etc. */
}
+```

int ret = SSL_connect(ssl);
if (ret <= 0) {
	int err = SSL_get_error(ssl, ret);
	if (err == SSL_ERROR_WANT_READ) {
		/* wait for socket readability */
	} else if (err == SSL_ERROR_WANT_WRITE) {
		/* wait for socket writability */
	} else {
		/* hard failure */
	}
}
+        /* wait for socket writability */
+    } else {
+        /* hard failure */
+    }
+}
long verify_res = SSL_get_verify_result(ssl);
if (verify_res != X509_V_OK) {
	/* reject connection */
}

X509 *peer = SSL_get1_peer_certificate(ssl);
if (peer == NULL) {
	/* server sent no certificate */
}

/* inspect subject, SANs, issuer, expiry if needed */
X509_free(peer);
- hostname match
- allowed key usage / EKU
- signature chain correctness

const char *req =
	"GET / HTTP/1.1\r\n"
	"Host: www.example.com\r\n"
	"Connection: close\r\n\r\n";

ret = SSL_write(ssl, req, (int)strlen(req));
if (ret <= 0) {
	int err = SSL_get_error(ssl, ret);
	/* handle retry or failure */
}
+    /* server sent no certificate */
+}
+
+/* inspect subject, SANs, issuer, expiry if needed */
+X509_free(peer);
+```

## Phase 6: Send Plaintext HTTP Request Through OpenSSL

After handshake, the application writes plaintext bytes. OpenSSL performs:

1. buffering / framing
2. TLS record creation
3. AEAD encryption
4. socket write of encrypted bytes

### C-Style Pseudo-Code: Send Request

```c
+const char *req =
+    "GET / HTTP/1.1\r\n"
+    "Host: www.example.com\r\n"
+    "Connection: close\r\n\r\n";
+
+ret = SSL_write(ssl, req, (int)strlen(req));
+if (ret <= 0) {
+    int err = SSL_get_error(ssl, ret);
+    /* handle retry or failure */
+}
+```

### What Happens Internally During `SSL_write`

Conceptually:

```text
plaintext HTTP bytes
	-> OpenSSL record layer fragments plaintext
	-> builds TLSInnerPlaintext
	-> derives per-record nonce from sequence number
	-> encrypts with negotiated AEAD
	-> serializes TLS record header + ciphertext
	-> calls socket write path
+```

## Phase 7: Read Plaintext HTTP Response via OpenSSL

Incoming kernel socket buffers contain encrypted TLS bytes. OpenSSL reads them, decrypts them, authenticates the records, and returns plaintext to the application.

### C-Style Pseudo-Code: Read Response

```c
char buf[8192];
for (;;) {
	ret = SSL_read(ssl, buf, sizeof(buf));
	if (ret > 0) {
		fwrite(buf, 1, (size_t)ret, stdout);
		continue;
	}

	int err = SSL_get_error(ssl, ret);
	if (err == SSL_ERROR_ZERO_RETURN) {
		break; /* close_notify */
	}
	if (err == SSL_ERROR_WANT_READ || err == SSL_ERROR_WANT_WRITE) {
		continue;
	}
	/* handle failure */
	break;
}
+```

## Full Practical Example: Minimal HTTPS Client Skeleton

```c
SSL_CTX *ctx = SSL_CTX_new(TLS_client_method());
SSL_CTX_set_verify(ctx, SSL_VERIFY_PEER, NULL);
SSL_CTX_set_default_verify_paths(ctx);

int fd = tcp_connect_hostname("www.example.com", "443");

SSL *ssl = SSL_new(ctx);
SSL_set_fd(ssl, fd);
SSL_set_tlsext_host_name(ssl, "www.example.com");
SSL_set1_host(ssl, "www.example.com");

if (SSL_connect(ssl) <= 0) {
	/* fail */
}

if (SSL_get_verify_result(ssl) != X509_V_OK) {
	/* fail */
}

const char *req =
	"GET / HTTP/1.1\r\n"
	"Host: www.example.com\r\n"
	"Connection: close\r\n\r\n";
SSL_write(ssl, req, (int)strlen(req));

char buf[8192];
int n;
while ((n = SSL_read(ssl, buf, sizeof(buf))) > 0) {
	consume_http_plaintext(buf, (size_t)n);
}

SSL_shutdown(ssl);
SSL_free(ssl);
close(fd);
SSL_CTX_free(ctx);
+```

## What Modern Applications Commonly Add on Top

Real production applications usually add:

1. Non-blocking event-loop integration
2. Connection pooling and session resumption
3. ALPN for HTTP/2 or HTTP/3 negotiation
4. Custom trust stores or certificate pinning
5. OCSP stapling and revocation policy
6. Telemetry around handshake failures and TLS versions/ciphers
7. Mutual TLS support when client certificates are required

## Reverse Proxy / Server Example

A TLS-terminating server or reverse proxy uses OpenSSL in the reverse direction:

- loads certificate chain and private key
- accepts TCP connections
- wraps each accepted socket in `SSL`
- calls `SSL_accept()` instead of `SSL_connect()`
- reads plaintext HTTP from `SSL_read()`
- writes plaintext HTTP responses via `SSL_write()`

### C-Style Pseudo-Code: OpenSSL Server Shape

```c
SSL_CTX *ctx = SSL_CTX_new(TLS_server_method());
SSL_CTX_use_certificate_chain_file(ctx, "server-chain.pem");
SSL_CTX_use_PrivateKey_file(ctx, "server-key.pem", SSL_FILETYPE_PEM);

int client_fd = accept(listen_fd, NULL, NULL);

SSL *ssl = SSL_new(ctx);
SSL_set_fd(ssl, client_fd);

if (SSL_accept(ssl) <= 0) {
	/* handshake failed */
}

char req_buf[8192];
int n = SSL_read(ssl, req_buf, sizeof(req_buf));
handle_http_request(req_buf, (size_t)n);
SSL_write(ssl, resp_buf, resp_len);
+```

## How OpenSSL Is Used in Modern Applications Conceptually

Three common integration models:

1. Direct blocking integration
- small tools, simple clients, demos

2. Non-blocking event-driven integration
- proxies, async clients, high-scale services

3. Wrapped by higher-level frameworks
- web servers, RPC stacks, SDKs, language runtimes

OpenSSL is often invisible to end users because many frameworks wrap it, but the core model remains the same: plaintext in at the application edge, encrypted records on the wire, plaintext back out after decryption.

## Debugging and Operational Signals

When using OpenSSL, practical debugging often starts with:

- `SSL_get_error()` after failed operations
- OpenSSL error queue (`ERR_get_error()`)
- peer certificate verification result
- negotiated protocol/cipher via `SSL_get_version()` and `SSL_get_cipher()`

### C-Style Pseudo-Code: Inspect Negotiated Session

```c
printf("TLS version: %s\n", SSL_get_version(ssl));
printf("Cipher: %s\n", SSL_get_cipher(ssl));
printf("ALPN: %s\n", SSL_get0_alpn_selected(...)); /* conceptual */
+```

## Why OpenSSL Still Matters

Even in modern environments with platform TLS APIs and alternative libraries (BoringSSL, NSS, wolfSSL, rustls), OpenSSL remains important because:

- many legacy and current servers still use it directly
- many higher-level runtimes link against it or API-compatible layers
- it remains a reference point for TLS integration patterns in C and C++ systems

---

**Navigation:**
- [<- Previous: TLS Record Layer and Packet Mapping](03-record-layer-and-packet-mapping.md)
- [Back to Index](README.md)
- [Next: OpenSSL vs Other TLS Libraries ->](05-openssl-vs-other-tls-libraries.md)
