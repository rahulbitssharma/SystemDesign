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
