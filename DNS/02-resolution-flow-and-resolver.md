# DNS — Resolution Flow and Resolver Role

## What Is a DNS Resolver?

A DNS resolver is the component that answers "what is the IP (or other record) for this name?"

There are two common resolver roles:

- **Stub resolver** (on host/OS): minimal client that forwards queries.
- **Recursive resolver** (in network): performs full lookup by querying other nameservers and caching results.

When people say "DNS resolver" in production architecture, they usually mean the **recursive resolver**.

## End-to-End Resolution (Typical)

1. App asks OS for `www.example.com`.
2. OS checks local sources (browser cache, OS cache, `/etc/hosts`).
3. On miss, OS stub sends query to configured recursive resolver.
4. Recursive resolver checks its cache.
5. On miss, recursive resolver asks a root server for `.com` delegation.
6. Resolver asks `.com` TLD server for `example.com` delegation.
7. Resolver asks authoritative server for `www.example.com` record.
8. Authoritative server replies (e.g. CNAME -> A/AAAA chain).
9. Recursive resolver caches each RRset by TTL and returns final answer to client.

## Query Types: Recursive vs Iterative

- **Recursive query**: Client asks resolver "give me final answer".
- **Iterative query**: Resolver asks nameservers and gets referrals until it reaches authority.

### Why It Is Called a Recursive Resolver

The resolver is called "recursive" because of the service contract it offers to clients, not because every upstream hop is recursive.

- Stub/host sends query with recursion desired.
- Recursive resolver takes full responsibility for returning a final answer (or final error).
- Internally, that resolver usually performs iterative queries to root, TLD, and authoritative servers.

So both statements are true at once: client->resolver interaction is recursive; resolver->upstream interaction is usually iterative.

## How Browser Requests Reach a DNS Resolver

When a browser needs `www.example.com`, name resolution usually flows like this:

1. Browser asks OS networking APIs to resolve the hostname (for example, via `getaddrinfo`-style calls).
2. OS stub resolver checks local sources/caches and then queries configured recursive resolver(s).
3. Recursive resolver returns answer to OS, and OS returns it to the browser.

Who configures which recursive resolver is used?

- Commonly the OS/network stack via DHCP-provided DNS servers from router/ISP, or manual/enterprise policy settings.
- Not directly "the internet" deciding per request.
- Browser code initiates lookup, but resolver choice is typically owned by OS/network configuration.

Important exception: if browser DNS-over-HTTPS is enabled, browser may send DNS directly to its configured DoH provider instead of OS-configured UDP/TCP resolver path.

## Is HTTPS Used for DNS?

Traditional DNS does not use HTTPS. Classic DNS typically uses:

- UDP/53 for most queries
- TCP/53 for specific cases (large responses, truncation fallback, zone transfer, reliability needs)

Modern encrypted DNS options:

- DNS-over-HTTPS (DoH): DNS messages carried inside HTTPS (HTTP/2 or HTTP/3) on port 443.
- DNS-over-TLS (DoT): DNS messages inside TLS on port 853.

So HTTPS can be used for DNS, but only when DoH is explicitly used by browser, OS, or resolver policy.

## How Certificates Are Exchanged (DoH/DoT)

Certificate exchange happens in the TLS handshake between DNS client and encrypted DNS endpoint.

High-level flow:

1. Client opens TLS connection to DoH/DoT server.
2. Server sends certificate chain (leaf + intermediates).
3. Client validates hostname, chain trust, validity period, and revocation/policy checks.
4. If valid, secure session keys are established.
5. DNS queries/responses flow inside that encrypted channel.

For DoH specifically, after TLS succeeds the client sends DNS payload as HTTP requests (often `application/dns-message` or JSON API variants).

## UDP vs TCP in DNS

UDP characteristics:

- Default for most DNS lookups due to low overhead and latency.
- One request/one response datagram model.
- No transport-level retransmission; retries/timeouts handled by DNS client logic.

TCP characteristics:

- Used when response is too large for acceptable UDP path limits and server sets truncation (`TC=1`).
- Required for AXFR/IXFR zone transfer operations.
- Useful when middleboxes fragment/drop UDP, or for stricter delivery semantics.

Typical resolver behavior:

- Try UDP first for normal queries.
- If truncated or policy requires, retry same query over TCP.

## How DNS Packets Look

Classic DNS message format (RFC-style wire layout):

```text
Header (12 bytes)
- ID
- Flags (QR, OPCODE, AA, TC, RD, RA, RCODE...)
- QDCOUNT, ANCOUNT, NSCOUNT, ARCOUNT

Question section
- QNAME, QTYPE, QCLASS

Answer section(s)
- NAME, TYPE, CLASS, TTL, RDLENGTH, RDATA

Authority section
- Usually NS/SOA data for referrals or negative answers

Additional section
- Extra helpful RRs (for example glue A/AAAA, OPT for EDNS)
```

Transport framing differences:

- UDP: exactly one DNS message per datagram.
- TCP: each DNS message is prefixed with a 2-byte length field on the stream.
- DoH: DNS message is payload inside HTTP over TLS, so HTTP and TLS frames carry it on the wire.

## Why Recursive Resolvers Are Critical

- Reduce latency through shared cache hits.
- Protect authoritative servers from repeated traffic.
- Apply policy/security (DNSSEC validation, filtering, split-horizon rules).
- Improve resilience using retries and alternate upstream paths.

## Failure Scenarios to Understand

- Resolver unavailable: apps fail to resolve names quickly.
- Stale cache after change: traffic may continue to old endpoint until TTL expiry.
- Bad delegation (NS/glue mismatch): domain partially or fully unresolvable.

---

**Navigation:**
- [<- Previous: Introduction](01-introduction.md)
- [Back to Index](README.md)
- [Next: Record Types and Format ->](03-record-types-and-format.md)
