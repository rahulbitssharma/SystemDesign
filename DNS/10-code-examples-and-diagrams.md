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
```

---

**Navigation:**
- [<- Previous: Consistency Guarantees](09-consistency.md)
- [Back to Index](README.md)
- [Next: Troubleshooting and Runbook ->](11-troubleshooting-and-runbook.md)
