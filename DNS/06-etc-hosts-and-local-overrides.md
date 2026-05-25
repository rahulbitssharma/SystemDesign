# DNS — /etc/hosts and Local Name Overrides

Before querying DNS, operating systems often check local name mappings.

On macOS/Linux, this file is usually:

```text
/etc/hosts
```

## What It Is

`/etc/hosts` is a static local mapping of hostname -> IP address.

Example:

```text
127.0.0.1   localhost
::1         localhost
127.0.0.1   api.internal.test
```

If `api.internal.test` exists in `/etc/hosts`, your machine can resolve it without contacting DNS.

## Where It Fits in Resolution

Typical order (can vary by OS config):

1. Browser/app cache
2. OS local sources (including `/etc/hosts`)
3. DNS query via stub resolver -> recursive resolver

## Why It Matters in Practice

- Local development environment overrides
- Emergency local pinning for troubleshooting
- Can cause confusing behavior if stale entries remain

## Operational Guidance

- Avoid long-term production dependence on `/etc/hosts`.
- Document temporary overrides and remove them after incident/debugging.
- Prefer controlled DNS records for shared environments.

---

**Navigation:**
- [<- Previous: Live Resolution Walkthrough](05-live-resolution-example.md)
- [Back to Index](README.md)
- [Next: Name Server Organization ->](07-nameserver-organization.md)
