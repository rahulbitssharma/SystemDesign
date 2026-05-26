# DNS — Code Examples and Diagrams

## Python: Basic Resolution Using System Resolver

```python
import socket

hostname = "www.github.com"

# Returns tuples for IPv4/IPv6 endpoints according to local resolver behavior.
results = socket.getaddrinfo(hostname, 443, proto=socket.IPPROTO_TCP)

ips = sorted({item[4][0] for item in results})
print(f"Resolved {hostname} to:")
for ip in ips:
    print(f"- {ip}")
```

This uses the OS resolver stack (which may include `/etc/hosts`, local cache, and configured recursive resolvers).

## Python: Query Specific Record Types (dnspython)

```python
# pip install dnspython
import dns.resolver

resolver = dns.resolver.Resolver()
for rdata in resolver.resolve("example.com", "A"):
    print("A:", rdata.to_text())

for rdata in resolver.resolve("google.com", "NS"):
    print("NS:", rdata.to_text())
```

## Pseudo-Code: Browser DNS Path Selection

```javascript
async function resolveForNavigation(hostname) {
  const cached = browserDnsCache.get(hostname);
  if (cached && !cached.isExpired()) return cached.addresses;

  const dohMode = settings.secureDnsMode; // off | automatic | strict
  const dohEndpoint = settings.dohEndpoint;

  if (dohMode !== "off") {
    const canUseDoh = await dohPolicyAllows(hostname, dohEndpoint);
    if (canUseDoh) {
      try {
        const answer = await resolveViaDoh(hostname, dohEndpoint);
        browserDnsCache.put(hostname, answer.addresses, answer.ttlSeconds);
        return answer.addresses;
      } catch (err) {
        if (dohMode === "strict") throw new Error("DoH failed in strict mode");
      }
    }
  }

  // Fallback path: use OS resolver APIs.
  const osAnswer = await osResolverGetAddrInfo(hostname);
  browserDnsCache.put(hostname, osAnswer.addresses, osAnswer.ttlSeconds);
  return osAnswer.addresses;
}
```

## Pseudo-Code: DoH Query and TLS Validation Flow

```javascript
async function resolveViaDoh(hostname, endpoint) {
  // TLS handshake happens during HTTPS connection setup.
  // Certificate validation includes hostname + trust chain checks.
  const queryWire = buildDnsWireQuery({ name: hostname, type: "A" });

  const resp = await fetch(endpoint, {
    method: "POST",
    headers: {
      "content-type": "application/dns-message",
      "accept": "application/dns-message"
    },
    body: queryWire
  });

  if (!resp.ok) throw new Error(`DoH failed: ${resp.status}`);

  const answerWire = new Uint8Array(await resp.arrayBuffer());
  return parseDnsWireResponse(answerWire);
}
```

## Sequence Diagram (Conceptual)

```text
Client/App
  |
  | query www.example.com
  v
OS Stub Resolver
  |
  | recursive query
  v
Recursive Resolver
  | cache miss
  +--> Root (.)          : where is .com?
  +<-- referral to TLD
  +--> .com TLD          : where is example.com?
  +<-- referral to auth NS
  +--> Authoritative NS  : A/AAAA for www.example.com?
  +<-- answer + TTL
  |
  +--> cache RRset
  v
Client gets final answer
```

## Architecture Diagram

```text
                   ┌──────────────────────────────┐
                   │  Authoritative DNS Provider  │
                   │ (many PoPs, anycast, HA)     │
                   └──────────────┬───────────────┘
                                  │
                     iterative DNS│
                                  v
┌──────────────┐   recursive   ┌────────────────────┐
│ Client Apps  │ ────────────> │ Recursive Resolver │
│ (Browsers,   │               │ (ISP/Public/Corp)  │
│ services)    │ <──────────── │ Shared cache       │
└──────┬───────┘    answers    └─────────┬──────────┘
       │                                  │
       │ local lookup order               │ iterative referrals
       v                                  v
┌────────────────┐                 Root -> TLD -> Auth
│ OS Stub +      │
│ /etc/hosts     │
└────────────────┘
```

## Useful Commands

```bash
# Local resolver configuration (macOS)
scutil --dns

# Follow full delegation path
dig +trace example.com A

# Show only final answers
dig www.github.com A +noall +answer

# Reverse lookup
dig -x 8.8.8.8 +noall +answer

# Force TCP instead of UDP
dig example.com A +tcp

# Show if UDP response was truncated (look for "tc" flag)
dig dnssec-failed.org DNSKEY +dnssec

# Query a DoH endpoint over HTTPS (wire transport is HTTPS/TLS)
curl -sS -H 'accept: application/dns-json' \
  'https://dns.google/resolve?name=example.com&type=A'
```

## Packet Shape Quick View

```text
UDP DNS packet:
[IP][UDP][DNS Header][Question][Answer...]

TCP DNS packet:
[IP][TCP][2-byte DNS length][DNS Header][Question][Answer...]

DoH packet stream:
[IP][TCP/QUIC][TLS][HTTP][DNS message payload]
```

---

**Navigation:**
- [<- Previous: Consistency Guarantees](09-consistency.md)
- [Back to Index](README.md)
- [Next: Troubleshooting and Runbook ->](11-troubleshooting-and-runbook.md)
