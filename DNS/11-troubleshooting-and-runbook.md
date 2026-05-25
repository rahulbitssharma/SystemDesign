# DNS — Troubleshooting and Runbook

This chapter focuses on practical debugging patterns for common DNS failures in production.

## High-Signal Failure Modes

1. **NXDOMAIN**
- Meaning: name does not exist.
- Common causes: typo, missing record, querying wrong zone, stale negative cache.

2. **NODATA (NoAnswer for queried type)**
- Meaning: name exists, but not for requested type (for example AAAA missing while A exists).
- Common causes: incomplete record rollout.

3. **SERVFAIL**
- Meaning: resolver could not complete lookup.
- Common causes: upstream timeout, DNSSEC validation failure, broken delegation chain.

4. **Timeout / No response**
- Meaning: network path/firewall/rate-limits or overloaded resolver/authoritative.

5. **Stale answer after change**
- Meaning: cached RRset still valid under old TTL.
- Common causes: changed record too soon, forgot to lower TTL before migration.

## Fast Triage Checklist

1. Verify local override first:

```bash
cat /etc/hosts
```

2. Check local resolver config (macOS):

```bash
scutil --dns
```

3. Compare default resolver vs known public resolvers:

```bash
dig api.example.com A +noall +answer
dig @1.1.1.1 api.example.com A +noall +answer
dig @8.8.8.8 api.example.com A +noall +answer
```

4. Walk delegation and detect breakpoints:

```bash
dig +trace api.example.com A
```

5. Inspect authoritative nameserver answers directly:

```bash
# Replace with your zone's authoritative NS
dig @ns1.example-dns-provider.net api.example.com A +noall +answer +authority
```

6. Check NS and glue consistency:

```bash
dig example.com NS +noall +answer
dig ns1.example.com A +noall +answer
```

7. Validate DNSSEC chain if enabled:

```bash
dig +dnssec example.com A
```

## Runbook: Zero-Downtime DNS Cutover

1. **T-24h (or old TTL window):** Lower TTL on target records.
2. **T-0:** Update record(s) to new endpoint(s).
3. **T+TTL window:** Keep old backend alive until old caches expire.
4. Verify from multiple networks/resolvers/regions.
5. Raise TTL back to normal after stability confirmed.

## Split-Horizon DNS Gotchas

If internal and external users see different answers:
- Check which resolver path each client uses.
- Confirm view/policy rules in enterprise DNS.
- Validate both internal and external authoritative data paths.

## Useful Command Patterns

```bash
# Follow CNAME chain and TTLs
dig www.example.com A +noall +answer

# Query specific record type and show authority
dig example.com MX +noall +answer +authority

# Reverse lookup for an IP
dig -x 203.0.113.10 +noall +answer
```

## Incident Notes Template

```text
Incident: DNS resolution failure for api.example.com
Start: 2026-05-25 10:15 UTC
Symptom: SERVFAIL from corp resolver, success from 1.1.1.1
Scope: Internal users only
Root cause: Broken DNSSEC validation after DS mismatch
Fix: Corrected DS at parent zone; flushed corp resolver cache
Prevention: Add pre-change DNSSEC validation checks in rollout pipeline
```

---

**Navigation:**
- [<- Previous: Code Examples and Diagrams](10-code-examples-and-diagrams.md)
- [Back to Index](README.md)
