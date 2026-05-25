# DNS — Introduction

DNS (Domain Name System) is the Internet's distributed naming system. It maps human-readable names like `api.example.com` to machine-usable data, most commonly IP addresses.

Without DNS, every client would need hardcoded IPs for every service, which would make failover, load balancing, and infrastructure migrations extremely hard.

## Why DNS Matters in System Design

- DNS is usually the first network dependency in a request path.
- DNS choices affect latency, availability, traffic steering, and blast radius.
- DNS caching behavior directly impacts rollout speed and rollback speed.

## DNS in One Picture

```text
User/App
  |
  | ask for api.example.com
  v
OS Stub Resolver
  |
  | recursive query
  v
Recursive Resolver (ISP/Public/Enterprise)
  |
  | iterative queries
  +--> Root Nameserver
  +--> TLD Nameserver (.com)
  +--> Authoritative Nameserver (example.com)
  |
  v
IP answer (A/AAAA) returned + cached with TTL
```

## Core Vocabulary

- **Domain name**: Hierarchical name, e.g. `www.example.com`.
- **Record (RR)**: Data mapped to a name, e.g. A, AAAA, CNAME, MX.
- **Zone**: Administrative portion of namespace (e.g. `example.com`).
- **Authoritative nameserver**: Source of truth for a zone.
- **Recursive resolver**: Resolver that asks other nameservers on your behalf.
- **TTL**: Time-to-live; maximum cache age for a record.

---

**Navigation:**
- [Back to Index](README.md)
- [Next: Resolution Flow and Resolver Role ->](02-resolution-flow-and-resolver.md)
