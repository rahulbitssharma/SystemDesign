# TLS — Troubleshooting and Runbook

This chapter is a practical troubleshooting guide for TLS failures in clients, servers, proxies, and service-to-service links.

## Failure Categories

Most TLS incidents fall into one of these buckets:

1. TCP connection never established
2. TLS handshake negotiation failure
3. Certificate/trust validation failure
4. Post-handshake record decryption failure
5. Application-protocol mismatch after TLS succeeds

## Fast Triage Flow

1. Confirm TCP connectivity first.
2. Confirm whether handshake started.
3. Identify the last successful TLS stage.
4. Separate certificate failures from cipher/version failures.
5. Separate TLS success from later HTTP/application failures.

## Common Failure Modes

### 1. Hostname Mismatch

Symptoms:

- handshake appears to complete or nearly complete
- client rejects peer certificate
- errors mention hostname, SAN, or certificate identity

Typical cause:

- requested hostname does not appear in certificate SAN list

### C-Style Pseudo-Code: Hostname Verification Failure Path

```c
if (!cert_verify_hostname(peer_leaf, expected_host)) {
    log_error("hostname mismatch: expected=%s", expected_host);
    tls_send_fatal_alert(session, TLS_ALERT_BAD_CERTIFICATE);
    tls_close(session);
}
```

### 2. Unknown CA / Trust Store Failure

Symptoms:

- `unknown_ca`
- verification result not OK
- chain cannot be built to a trusted root

Typical causes:

- missing private root in enterprise environment
- server forgot intermediate certificate
- wrong trust store on client

### 3. Protocol Version Mismatch

Symptoms:

- `protocol_version`
- `handshake_failure`
- connection closes immediately after ClientHello / ServerHello attempt

Typical causes:

- server only allows TLS 1.3, client only offers TLS 1.2
- policy disables older versions

### 4. Cipher / Group Negotiation Failure

Symptoms:

- `handshake_failure`
- HelloRetryRequest loops or group mismatch failures

Typical causes:

- no overlapping cipher suites
- no overlapping ECDHE groups
- policy restrictions on one side

### 5. Expired Certificate

Symptoms:

- certificate validation failure
- time-validity errors in client logs

Typical causes:

- cert renewal failed
- deployment still points to old chain
- system clock skew

### 6. Bad Record MAC / Decrypt Error

Symptoms:

- `bad_record_mac`
- `decrypt_error`
- connection dies after handshake, often during application traffic

Typical causes:

- key mismatch
- corrupted traffic
- broken offload path
- protocol or implementation bug

## Stage-by-Stage Debugging Map

| Last Successful Stage | Likely Problem Area |
|---|---|
| TCP connect only | firewall, routing, wrong port, listener down |
| ClientHello sent | version/cipher/group mismatch, SNI routing issue |
| ServerHello received | cert chain or later handshake validation |
| Certificate received | trust store, hostname, EKU, expiry |
| Finished exchanged | post-handshake record or application protocol problem |
| HTTP request sent | application-level issue above TLS |

## Practical Command-Line Checks

### OpenSSL s_client

Useful for seeing handshake and certificate details:

```bash
openssl s_client -connect www.example.com:443 -servername www.example.com
```

Useful variants:

```bash
openssl s_client -connect www.example.com:443 -servername www.example.com -showcerts
openssl s_client -connect www.example.com:443 -servername www.example.com -tls1_3
openssl s_client -connect www.example.com:443 -servername www.example.com -alpn h2
```

### curl

Useful for application-level confirmation over HTTPS:

```bash
curl -v https://www.example.com/
```

### Packet Capture

Use packet capture to answer:

- did TCP connect?
- was ClientHello sent?
- did any TLS alert return?
- is there encrypted application data after handshake?

## Runbook: Certificate Incident

When a production certificate incident occurs:

1. Identify affected hostname(s).
2. Inspect currently served leaf and intermediates.
3. Check expiry time and SAN coverage.
4. Check whether correct chain file is deployed.
5. Verify trust path from representative clients.
6. Roll forward renewed certificate/chain.
7. Re-test with `openssl s_client` and application-level probes.

### C-Style Pseudo-Code: Inspect Verify Result in Client/Proxy

```c
long verify_res = SSL_get_verify_result(ssl);
if (verify_res != X509_V_OK) {
    log_error("tls verify failed: code=%ld", verify_res);
    X509 *peer = SSL_get1_peer_certificate(ssl);
    dump_peer_cert_summary(peer);
    X509_free(peer);
}
```

## Runbook: mTLS Failure

Typical mTLS problem sources:

- client cert missing
- wrong client CA bundle on server
- EKU does not allow client auth
- private key/cert mismatch

### C-Style Pseudo-Code: Server-Side mTLS Setup

```c
SSL_CTX_set_verify(ctx, SSL_VERIFY_PEER | SSL_VERIFY_FAIL_IF_NO_PEER_CERT, NULL);
SSL_CTX_load_verify_locations(ctx, "/etc/tls/client-ca.pem", NULL);
```

### Triage Questions

1. Did the server request a client certificate?
2. Did the client send one?
3. Was the client chain trusted by server?
4. Was client EKU valid for TLS client auth?

## Runbook: TLS Succeeds but App Fails

Sometimes TLS is healthy but the application still breaks.

Examples:

- ALPN negotiated `http/1.1` when client expected `h2`
- proxy routes wrong host after successful TLS termination
- server closes due to malformed HTTP request

### C-Style Pseudo-Code: Log Negotiated Session

```c
log_info("tls_version=%s", SSL_get_version(ssl));
log_info("cipher=%s", SSL_get_cipher(ssl));
```

## Runbook: Non-Blocking/OpenSSL Event-Loop Bugs

Symptoms:

- handshake appears stuck
- application spins CPU
- partial writes or reads not handled properly

Cause pattern:

- code ignores `SSL_ERROR_WANT_READ` / `SSL_ERROR_WANT_WRITE`

### Correct Handling Shape

```c
int ret = SSL_connect(ssl);
if (ret <= 0) {
    int err = SSL_get_error(ssl, ret);
    if (err == SSL_ERROR_WANT_READ) {
        wait_for_readable(fd);
    } else if (err == SSL_ERROR_WANT_WRITE) {
        wait_for_writable(fd);
    } else {
        fail_connection();
    }
}
```

## Logging Recommendations

For TLS-aware services and proxies, log at least:

- SNI / requested host
- peer address
- TLS version
- cipher suite
- verification result
- alert/failure category
- client-cert subject for mTLS paths

## Incident Checklist

1. Reproduce with `openssl s_client`.
2. Check cert chain and hostname.
3. Check negotiated version/cipher/policy.
4. Look for TLS alerts versus TCP resets.
5. Confirm whether failure is pre-handshake, in-handshake, or post-handshake.
6. Confirm whether issue is TLS or HTTP/application behavior above TLS.

---

**Navigation:**
- [<- Previous: OpenSSL vs Other TLS Libraries](05-openssl-vs-other-tls-libraries.md)
- [Back to Index](README.md)
