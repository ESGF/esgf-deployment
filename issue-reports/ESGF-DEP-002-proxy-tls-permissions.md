# ESGF-DEP-002 — Proxy permissions task assumes TLS files exist

## Summary

- **Classification:** Configuration bug.
- **Observed scope:** Reproduced on Rocky Linux 8.10 and 9.8 when TLS
  certificate and key were omitted.
- **Status:** Candidate fix prepared on local branch
  `fix/ESGF-DEP-002-proxy-tls-guard`, commit `32c6b13`; syntax check passes.
  The commit is local and unpushed; deployment testing is pending.

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

The historical Rocky 8 and Rocky 9 FUTURA proxy overrides run the permissions
task only when both `ssl_certificate` and `ssl_private_key` are defined.

The upstream role should make the permission operation conditional on the
TLS configuration and, ideally, on the relevant files being present. This
preserves the HTTP-only setup while retaining correct ownership and
permissions when TLS is enabled.

The candidate branch applies the same condition used by the certificate
installation task: both `ssl_certificate` and `ssl_private_key` must be
defined before attempting to change file permissions.

The successful earlier deployments used temporary Rocky 8 and Rocky 9
FUTURA overrides, which have since been removed. Their behavior and test
results are summarized above; the committed candidate fix has not yet been
deployed.

## Verification and limits

With the historical local guards, HTTP-only deployment completed and the
proxy started on both tested OSes. HTTPS and certificate provisioning were
not tested. The candidate branch has not been deployed; validate both
HTTP-only and TLS-enabled configurations before treating the issue as fixed.
