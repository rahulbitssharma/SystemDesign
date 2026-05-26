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

## DoH vs DoT: Key Differences

Both encrypt DNS, but they differ in encapsulation and operational behavior.

| Dimension | DoH | DoT |
|---|---|---|
| Transport | HTTPS (HTTP/2 or HTTP/3) | TLS over TCP |
| Typical port | 443 | 853 |
| Payload shape | DNS message inside HTTP request/response | Raw DNS message inside TLS stream |
| Visibility to middleboxes | Looks like general HTTPS traffic | Clearly DNS-over-TLS traffic |
| Browser adoption | Very common in modern browsers | Less common directly in browsers |
| Common use | Browser secure DNS, privacy over web path | OS/resolver-to-resolver encrypted DNS |

Example choice in practice:

- Browser wants DNS privacy and easy deployment through existing HTTPS infrastructure: uses DoH endpoint like `https://dns.example.net/dns-query`.
- Enterprise resolver encrypts upstream DNS between resolvers with explicit DNS transport identity: often uses DoT on port 853.

## Packet Flow Example: DoH vs DoT

DoH flow (simplified):

1. Client -> DoH server: TCP or QUIC connect (usually 443)
2. TLS handshake + certificate validation
3. HTTP request carrying DNS query bytes
4. HTTP response carrying DNS answer bytes

DoT flow (simplified):

1. Client -> DoT server: TCP connect on 853
2. TLS handshake + certificate validation
3. DNS query bytes over TLS stream (no HTTP layer)
4. DNS response bytes over TLS stream

## Is DoH Default in Modern Browsers?

Short answer: often "automatic by default" in modern browsers, but not always "strictly on" for every network.

Typical current behavior:

- Browser enables secure-DNS auto-upgrade mode for many users.
- Browser first checks if known DoH can be used for the current resolver/provider.
- If compatible, browser upgrades DNS transport to DoH.
- If not compatible and strict mode is not enabled, browser falls back to system DNS.

So, in modern browsers DoH support is common and frequently enabled in auto mode, while mandatory DoH-only behavior is less common.

At OS level, encrypted DNS support is growing, but default behavior varies by platform and enterprise policy.

## How Certificates Are Exchanged (DoH/DoT)

Certificate exchange happens in the TLS handshake between DNS client and encrypted DNS endpoint.

High-level flow:

1. Client opens TLS connection to DoH/DoT server.
2. Server sends certificate chain (leaf + intermediates).
3. Client validates hostname, chain trust, validity period, and revocation/policy checks.
4. If valid, secure session keys are established.
5. DNS queries/responses flow inside that encrypted channel.

For DoH specifically, after TLS succeeds the client sends DNS payload as HTTP requests (often `application/dns-message` or JSON API variants).

## What Browser System Calls Look Like When DoH Is Enabled

Conceptually, browser networking code chooses one of two resolution paths.

Path A: DoH enabled and usable

1. Browser checks host resolver/cache and its own DNS cache.
2. Browser opens HTTPS connection to DoH endpoint.
3. Browser validates DoH server certificate in TLS handshake.
4. Browser sends DNS query in HTTP request body/URL.
5. Browser parses DNS response and caches per TTL.
6. Browser opens TCP/TLS (or QUIC/TLS) connection to destination origin using resolved IP.

Path B: DoH disabled or unavailable

1. Browser calls OS resolver API (for example, `getaddrinfo`-style API).
2. OS stub sends DNS query to configured recursive resolver over UDP/TCP 53 (or OS-configured encrypted transport).
3. OS returns results to browser.
4. Browser connects to destination origin.

Important nuance: exact low-level system calls differ across engines and platforms, but this decision split (browser DoH path vs OS resolver path) is the key architecture.

## How HTTPS Works Over TCP (And What OS Does)

At a high level, HTTPS is HTTP carried inside TLS, and TLS is carried over TCP (or over QUIC for HTTP/3). For HTTP/1.1 and HTTP/2, the usual stack is:

`HTTP plaintext` -> `TLS records (encrypted)` -> `TCP segments` -> `IP packets` -> `Link frames`

Detailed sequence for HTTPS over TCP:

1. Application (browser) asks OS to create a socket (`socket`).
2. Application asks OS to connect TCP to remote IP:443 (`connect`).
3. OS TCP stack performs 3-way handshake (SYN, SYN-ACK, ACK).
4. After TCP is established, browser TLS stack sends `ClientHello` bytes on that socket.
5. Server replies with handshake messages including certificate chain.
6. Browser validates certificate and completes key exchange.
7. TLS session keys are installed in browser TLS state.
8. Browser writes HTTP request plaintext to TLS library.
9. TLS library encrypts and emits TLS records; OS sends ciphertext bytes via TCP/IP.
10. On receive path, OS delivers ciphertext bytes from socket to browser process; TLS library decrypts and returns plaintext HTTP response to browser networking code.

Who implements what:

- OS kernel: sockets API, TCP state machine, packet transmit/receive, congestion control.
- Browser/app TLS stack (or platform TLS library): handshake logic, certificate validation, key schedule, record encryption/decryption.
- Application protocol layer: HTTP semantics (methods, headers, bodies).

## HTTPS Packet Format (Conceptual)

For HTTP/2 over TLS over TCP, packets are conceptually layered as:

```text
L2 Frame
	L3: IP header
		L4: TCP header
			TLS record header + encrypted payload
				(inside decrypted payload: HTTP/2 frames)
```

Important note: OS does not usually deliver decrypted HTTPS to apps. It delivers socket bytes (ciphertext after TLS starts). Decryption is typically done in user-space TLS library inside browser/app process. After decryption, app sees plaintext HTTP messages.

## Connecting HTTPS Internals to DoH

DoH is simply DNS messages carried as an HTTPS application payload.

- Normal HTTPS website: HTTPS payload is HTTP content API/page traffic.
- DoH HTTPS session: HTTPS payload is DNS wire-format query/response.

So DoH reuses the same HTTPS/TLS/TCP machinery, but with DNS semantics in the payload.

## Detailed Resolution Flow with Browser DoH

When browser-side DoH is active, lookup and page load usually look like this:

1. User enters `https://www.example.com`.
2. Browser checks local caches (host cache, DNS cache, preconnect/prefetch state).
3. Browser sends DoH query for A/AAAA to configured DoH endpoint.
4. DoH endpoint's recursive resolver performs iterative lookup if cache miss.
5. DoH response returns answer and TTL to browser.
6. Browser chooses endpoint candidate(s) (IPv6/IPv4 policy such as Happy Eyeballs).
7. Browser connects to target IP and starts HTTPS handshake with target website.
8. Website certificate exchange happens separately from DoH certificate exchange.
9. HTTP request/response for page content proceeds after website TLS is established.

Two separate TLS contexts exist:

- TLS session A: browser <-> DoH server (for DNS transport privacy)
- TLS session B: browser <-> website origin (for application content security)

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
