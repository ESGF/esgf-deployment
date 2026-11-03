# ESGF-DEP-006 — Initial check-mode run cannot handle an absent Docker daemon

## Summary

- **Classification:** Check-mode validation limitation.
- **Observed scope:** Rocky Linux 8.10 before Docker installation; Rocky
  Linux 9.8 was not tested for this behavior.
- **Status:** Reproduced; retained as a check-mode limitation, with no
  issue-specific fix branch. Skipping Docker API tasks would omit deployment
  operations from the dry-run rather than validate them.

## Evidence and cause

On the fresh Rocky Linux 8.10 VM, the initial Ansible check-mode run stopped
in the auth role's uninstall tasks while querying the Docker API. The Docker
daemon was not available.

Check mode predicts changes instead of executing preceding installation and
startup tasks. However, the auth uninstall tasks still invoke Docker
container modules, which require a working daemon. As a result, a first-run
dry run cannot safely simulate the entire playbook on a host without Docker.

## Workaround and proposed improvement

The deployment was run normally to install and start Docker before proceeding.
For future check-mode runs, first ensure Docker is installed and running. No
code change has been prepared because the safe workaround is to run check mode
only when the daemon is available.

An upstream improvement would guard Docker API operations in check mode when
the daemon is unavailable, or restructure the role so a fresh-host check run
does not query containers before Docker is installed. Any such change should
preserve meaningful dry-run validation of the remaining tasks.

## Verification and limits

The initial check-mode failure was observed and the subsequent real
deployment completed. A second check-mode run after Docker installation was
not performed. Rocky Linux 9.8 behavior is unknown.
