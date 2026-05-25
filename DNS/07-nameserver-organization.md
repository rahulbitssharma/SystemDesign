# DNS — How Name Servers Are Organized

DNS is a distributed hierarchy, not a central database.

## Layers of Name Servers

1. **Root name servers**
- Top of DNS hierarchy (`.` zone).
- 13 logical server identities (`a.root-servers.net` to `m.root-servers.net`).
- Implemented via many physical instances globally (anycast).

2. **TLD name servers**
- Serve top-level domains like `.com`, `.org`, `.io`, country TLDs.
- Return delegation (`NS`) to authoritative servers for second-level domains.

3. **Authoritative name servers**
- Hold source-of-truth records for a zone (e.g. `example.com`).
- Return final records (A/AAAA/CNAME/TXT/MX/etc.) or authoritative negative responses.

4. **Recursive resolvers**
- Not authoritative for most public domains.
- Perform lookups and caching for clients.

## Delegation and Glue

When parent zone delegates child zone via NS records, child NS names may need "glue" A/AAAA records in parent to break circular dependencies.

Example concept:

```text
example.com.      NS   ns1.example.com.
ns1.example.com.  A    203.0.113.53   ; glue in parent if required
```

## Diagram: Hierarchy

```text
                  [ Root Zone (.) ]
                         |
                         v
                     [ .com TLD ]
                         |
                         v
             [ Authoritative for example.com ]
                         |
                         v
                answer: www.example.com A ...
```

---

**Navigation:**
- [<- Previous: /etc/hosts and Local Overrides](06-etc-hosts-and-local-overrides.md)
- [Back to Index](README.md)
- [Next: How DNS Scales ->](08-scaling.md)
