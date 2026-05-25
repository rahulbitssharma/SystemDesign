# DNS — Record Types and Format

DNS data is stored as **resource records** (RRs) in RRsets. A practical way to view record shape is:

```text
NAME    TTL    CLASS   TYPE    RDATA
```

Usually `CLASS` is `IN` (Internet).

## Common Record Types

- **A**: Name -> IPv4 address
- **AAAA**: Name -> IPv6 address
- **CNAME**: Name alias to another canonical name (cannot coexist with other data at same owner name)
- **NS**: Delegates zone/subzone to nameservers
- **SOA**: Start of authority; zone metadata (primary NS, serial, refresh/retry/expire, negative TTL)
- **MX**: Mail exchanger host(s) for domain
- **TXT**: Arbitrary text (SPF, verification, DKIM fragments)
- **SRV**: Service discovery with priority/weight/port/target
- **PTR**: Reverse DNS (IP -> name)
- **CAA**: Which certificate authorities may issue TLS certs

## Deep Dive: CNAME, NS, and MX

### CNAME (Canonical Name)

- A `CNAME` makes one name an alias of another name.
- Resolver behavior: when it gets a CNAME answer, it restarts lookup for the target name and then returns the final A/AAAA (or other requested type) answer.
- Constraint: an owner name with `CNAME` cannot also have other records like `A`, `AAAA`, `MX`, or `TXT` at that same owner name.
- Operational note: this is why zone apex (`example.com`) is usually not a CNAME in classic DNS setups; apex often needs `NS`, `SOA`, and sometimes `MX`.

Example:

```dns
www     IN CNAME app-lb.example.net.
```

`www.example.com` is an alias, and `app-lb.example.net` holds the real address records.

### NS (Name Server)

- `NS` records define which authoritative nameservers serve a zone.
- At parent/child boundaries, NS records represent delegation (for example, `.com` delegates `example.com` to that domain's authoritative NS set).
- Inside a zone apex, NS records publish that zone's authoritative servers.
- If NS hostnames are inside the delegated child zone, parent zone often must include glue A/AAAA records.

Example:

```dns
@       IN NS   ns1.example.com.
@       IN NS   ns2.example.com.
```

### MX (Mail Exchanger)

- `MX` records tell senders where to deliver email for a domain.
- Each MX has a preference number; lower value means higher priority.
- Mail transfer agents try the lowest-preference reachable target first, then fail over to higher numbers.
- MX targets should resolve to A/AAAA records (not CNAME targets), to avoid ambiguous delivery behavior.

Example:

```dns
@       IN MX 10 mail1.example.com.
@       IN MX 20 mail2.example.com.
```

## What Is a Zone Snippet?

A zone snippet is a partial text extract from a zone's authoritative data, usually written in zone-file syntax. It is not a separate DNS concept on the wire; it is just a human-editable representation of DNS records.

In practice, the same logical zone data can be stored in different backends:

- Flat zone files (common in BIND-style authoritative setups)
- Databases (SQL/NoSQL) in managed DNS/control-plane systems
- In-memory/generated data structures in custom authoritative services

So, a "zone snippet" is typically shown as text, but production storage may or may not be a literal file.

## Example Zone Snippet

```dns
$ORIGIN example.com.
$TTL 300

@       IN SOA ns1.example.com. dns-admin.example.com. (
            2026052501 ; serial
            3600       ; refresh
            600        ; retry
            1209600    ; expire
            300        ; negative cache TTL
)

@       IN NS   ns1.example.com.
@       IN NS   ns2.example.com.

@       IN A    203.0.113.10
www     IN CNAME app-lb.example.net.
api     IN A    203.0.113.20
@       IN MX 10 mail1.example.com.
@       IN TXT  "v=spf1 include:_spf.example.net -all"
```

## How Records Look in `dig` Output

```text
www.github.com.   2925   IN   CNAME   github.com.
github.com.         13   IN   A       140.82.114.4
```

Interpretation:
- `www.github.com.` is an alias with TTL 2925 seconds.
- Resolver must also fetch `A` for `github.com.` (TTL 13 seconds in that sample).

---

**Navigation:**
- [<- Previous: Resolution Flow and Resolver Role](02-resolution-flow-and-resolver.md)
- [Back to Index](README.md)
- [Next: Caching in DNS ->](04-caching.md)
