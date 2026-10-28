# ESGF-DEP-002 — Proxy permissions task assumes TLS files exist

## Summary

- **Classification:** Configuration bug.
- **Observed scope:** Reproduced on Rocky Linux 8.10 and 9.8 when TLS
  certificate and key were omitted.
- **Status:** OS-specific FUTURA workarounds are applied; an upstream fix is
  proposed.

## Evidence and cause

The HTTP-only deployments omitted the TLS certificate and private key. The
proxy role conditionally installs those files but then executes a permissions
task against their paths unconditionally. The task fails when those paths do
not exist.

On Rocky Linux 9.8, this appeared after the pip prerequisite had been fixed:
the intermediate run failed in the upstream proxy role with
`ok=46`, `changed=22`, `failed=1`, `skipped=29`. The same issue was observed
on Rocky Linux 8.10 with TLS omitted.

## Fixes and proposed upstream change

The Rocky 8 and Rocky 9 FUTURA proxy overrides run the permissions task only
when both `ssl_certificate` and `ssl_private_key` are defined.

The upstream role should make the permission operation conditional on the
TLS configuration and, ideally, on the relevant files being present. This
preserves the HTTP-only setup while retaining correct ownership and
permissions when TLS is enabled.

FUTURA overrides:

- [`Rocky 8 proxy role`](../ansible/overrides/rocky8/roles/proxy/tasks/main.yml)
- [`Rocky 9 proxy role`](../ansible/overrides/rocky9/roles/proxy/tasks/main.yml)

## Verification and limits

With the guards, HTTP-only deployment completed and the proxy started on both
tested OSes. HTTPS and certificate provisioning were not tested, so those
paths still require validation after an upstream change.
