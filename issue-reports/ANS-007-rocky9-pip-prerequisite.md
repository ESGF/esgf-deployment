# ANS-007 — Rocky Linux 9 lacks the pip prerequisite for the Docker SDK

## Summary

- **Classification:** Platform compatibility/deployment bug.
- **Observed scope:** Rocky Linux 9.8; not observed on Rocky Linux 8.10.
- **Status:** Rocky 9-specific FUTURA workaround applied; upstream fix
  proposed.

## Evidence and cause

The clean Rocky Linux 9.8 upstream-only run failed at the Docker role's
`Install Docker SDK` task with:

```text
Unable to find any of pip3 to use. pip needs to be installed.
```

Docker CE packages and the upstream Docker mount drop-in had been installed,
but pip had not. The upstream role's pip setup covered CentOS/RHEL and
AlmaLinux, but not Rocky Linux. On Rocky Linux 9, `python3-pip` provides the
required Python 3 pip executable.

The baseline recap was `ok=8`, `changed=6`, `failed=1`, `skipped=3`. The
Rocky Linux 8.10 setup passed this task, so the missing package was not
observed there.

## Fixes and proposed upstream change

The Rocky 9 FUTURA Docker role override installs `python3-pip` before invoking
the Ansible `pip` module. It retains the upstream mount-flags behavior; the
Rocky 8 mount workaround is not included.

The upstream role should include Rocky Linux in pip prerequisite handling and
install the appropriate distribution package before installing the Docker
SDK. Use distribution facts/package-manager variables where possible so the
task remains correct across supported RHEL-family variants.

FUTURA override: [`Rocky 9 Docker role`](../ansible/overrides/rocky9/roles/docker/tasks/main.yml).

## Verification and limits

After adding only the pip prerequisite, the play passed Docker setup and
exposed ANS-002 in the proxy role (`ok=46`, `changed=22`, `failed=1`,
`skipped=29`). With the pip fix and Rocky 9 no-TLS proxy guard, deployment
completed with `ok=50`, `changed=13`, `unreachable=0`, `failed=0`,
`skipped=30`.
