# ESGF-DEP-007 — Rocky Linux 9 lacks the pip prerequisite for the Docker SDK

## Summary

- **Classification:** Platform compatibility/deployment bug.
- **Observed scope:** Rocky Linux 9.8; not observed on Rocky Linux 8.10.
- **Status:** Candidate fix prepared on local branch
  `fix/ESGF-DEP-007-rocky9-pip`, commit `4ae60b8`; syntax check passes.
  The commit is local and unpushed; deployment testing is pending.

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

The clean Rocky Linux 9.8 data-serving baseline on 2026-10-30 reproduced the
same failure with `ok=8`, `changed=6`, `unreachable=0`, `failed=1`,
`skipped=3`, before any service containers were configured.

## Fixes and proposed upstream change

The candidate branch installs `python3-pip` on Rocky Linux before invoking
the Ansible `pip` module. The earlier successful tests used the historical
Rocky 9 FUTURA Docker role override, which retained the upstream mount-flags
behavior; the Rocky 8 mount workaround was not included.

The upstream role should include Rocky Linux in pip prerequisite handling and
install the appropriate distribution package before installing the Docker
SDK. Use distribution facts/package-manager variables where possible so the
task remains correct across supported RHEL-family variants.

The successful earlier Rocky 9 tests used a temporary FUTURA Docker role
override, which has since been removed. The candidate fix restores the
missing prerequisite in the fork branch.

## Verification and limits

After adding only the pip prerequisite, the play passed Docker setup and
exposed ESGF-DEP-002 in the proxy role (`ok=46`, `changed=22`, `failed=1`,
`skipped=29`). With the pip fix and Rocky 9 no-TLS proxy guard, deployment
completed with `ok=50`, `changed=13`, `unreachable=0`, `failed=0`,
`skipped=30`.

In the Rocky Linux 9.8 data-serving test on 2026-10-30, the same Rocky 9
overrides completed the full deployment with `ok=50`, `changed=26`,
`unreachable=0`, `failed=0`, `skipped=30`; the synthetic dataset was served
through THREDDS and fileserver. The higher changed count includes configuring
the data-serving scenario. The candidate branch has not yet been run on
Rocky Linux 9.
