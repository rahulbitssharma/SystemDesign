# DNS — Live Resolution Walkthrough

Below are real command samples (captured on May 25, 2026) showing how DNS resolution and delegation look in practice.

## 1. CNAME -> A Chain Example

Command:

```bash
dig www.github.com A +noall +answer
```

Observed output:

```text
www.github.com.         2925    IN      CNAME   github.com.
github.com.             13      IN      A       140.82.114.4
```

What happened:
1. `www.github.com` is an alias.
2. Resolver followed CNAME to `github.com`.
3. Final A record returned as `140.82.114.4`.
4. Different records in chain can have different TTLs.

## 2. Full Delegation Walk (`+trace`)

Command:

```bash
dig +trace example.com A
```

Key excerpts:

```text
.       NS  a.root-servers.net. ... m.root-servers.net.
com.    NS  a.gtld-servers.net. ... m.gtld-servers.net.
example.com. NS hera.ns.cloudflare.com.
example.com. NS elliott.ns.cloudflare.com.
example.com. 300 IN A 104.20.23.154
example.com. 300 IN A 172.66.147.243
```

Interpretation:
1. Root tells resolver where `.com` is served.
2. `.com` TLD tells resolver who is authoritative for `example.com`.
3. Authoritative Cloudflare nameserver returns final A records.

## 3. NS Delegation Data

Commands:

```bash
dig com NS +noall +answer
dig google.com NS +noall +answer
```

Sample output includes:

```text
com.        172800 IN NS  a.gtld-servers.net.
...
google.com. 257531 IN NS  ns1.google.com.
google.com. 257531 IN NS  ns2.google.com.
```

This is how hierarchy and delegation are encoded in DNS.

## Latency and Caching Observation

- First lookup after cache miss is slower due to iterative work.
- Subsequent lookups are usually faster due to resolver cache hits.
- Any record change remains globally mixed until old TTL windows expire.

## Helpful Debug Commands

```bash
# Ask your default resolver
nslookup api.example.com

# Query a specific resolver directly
# (example: 1.1.1.1)
dig @1.1.1.1 api.example.com A +noall +answer

# Show answer + authority section
dig api.example.com A +noall +answer +authority
```

---

**Navigation:**
- [<- Previous: Caching](04-caching.md)
- [Back to Index](README.md)
- [Next: /etc/hosts and Local Overrides ->](06-etc-hosts-and-local-overrides.md)
