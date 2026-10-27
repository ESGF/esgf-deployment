# ESGF Ansible Deployment Issue Log

**Last updated:** 2026-10-27
**Scope:** Tests and findings from deploying the upstream `esgf-docker`
Ansible playbook.

## 1. Tested environments

| OS | Image | Environment | Docker | Ansible Core | Upstream | Configuration |
|---|---|---|---|---|---|---|
| Rocky Linux 8.10 | `RockyLinux8.10` (`fd621c1a-6a3b-44dd-9c62-2390551bc7ec`) | IFCA OpenStack, `cm14-compute-8` | 26.1.3 | 2.16.3 | `esgf-docker` `5e2f3c3` | `esgfdeploy/*:latest`; HTTP only; no datasets |
| Rocky Linux 9.8 | `RockyLinux9.8-GenericCloud-Base` (`cc0c5ae4-d593-423e-98e9-71a934c2d6bf`) | IFCA OpenStack, `cm14-compute-8` | 29.9.2 | 2.16.3 | `esgf-docker` `5e2f3c3` | `esgfdeploy/*:latest`; HTTP only; no datasets |

Both OS tests began with clean VMs and upstream roles. The reproducible
baseline-first procedure is in the
[`Ansible test methodology`](./ansible-test-methodology.md).

## 2. Findings

The OS scope below describes where each finding was observed; it does not
establish that a finding cannot occur on other systems.

| ID | Finding | Observed scope | Status | Detailed report |
|---|---|---|---|---|
| ANS-001 | THREDDS container startup fails with upstream `MountFlags=shared` | Rocky Linux 8.10 | Rocky 8 workaround applied | [ANS-001](./issue-reports/ANS-001-mount-flags-container-startup.md) |
| ANS-002 | Proxy permission task fails when TLS files are omitted | Rocky Linux 8.10 and Rocky Linux 9.8 | OS-specific workarounds applied; upstream fix proposed | [ANS-002](./issue-reports/ANS-002-proxy-tls-permissions.md) |
| ANS-003 | Unchanged playbook rerun recreates service containers | Rocky Linux 8.10 and Rocky Linux 9.8 | Reproduced; upstream fix proposed, not applied | [ANS-003](./issue-reports/ANS-003-container-recreation.md) |
| ANS-004 | Duplicate `data_datasets` key in upstream example variables | OS-independent (source inspection) | Confirmed by inspection; runtime behavior untested | [ANS-004](./issue-reports/ANS-004-duplicate-data-datasets.md) |
| ANS-005 | Futura-local playbook does not discover upstream sibling `group_vars` | OS-independent (test setup) | Documented caveat; no group variables loaded | [ANS-005](./issue-reports/ANS-005-group-vars-discovery.md) |
| ANS-006 | Initial check-mode run requires an unavailable Docker daemon | Rocky Linux 8.10 | Reproduced; no code change | [ANS-006](./issue-reports/ANS-006-check-mode-docker-daemon.md) |
| ANS-007 | Rocky 9 host lacks pip prerequisite for Docker SDK | Rocky Linux 9.8 | Rocky 9 workaround applied; upstream fix proposed | [ANS-007](./issue-reports/ANS-007-rocky9-pip-prerequisite.md) |

## 3. Deployment verification

| OS | Clean upstream-only baseline | With OS-specific fixes | Repeat run |
|---|---|---|---|
| Rocky Linux 8.10 | Failed at THREDDS startup: `ok=19`, `changed=11`, `failed=1`, `skipped=11` | Passed: `ok=49`, `changed=18`, `failed=0`, `skipped=29` | Passed: `ok=47`, `changed=6`, `failed=0`, `skipped=31`; all six service container IDs changed |
| Rocky Linux 9.8 | Failed at Docker SDK installation: `ok=8`, `changed=6`, `failed=1`, `skipped=3` | Passed after pip and no-TLS fixes: `ok=50`, `changed=13`, `failed=0`, `skipped=30` | Passed: `ok=50`, `changed=10`, `failed=0`, `skipped=30`; all six service container IDs changed |

After applying the respective fixes, `proxy`, `thredds`, `fileserver`,
`solr-master`, `solr-slave`, and `search` ran on both operating systems. The
`/health`, THREDDS catalog, ESGF search query, and Solr replication endpoints
for `files`, `datasets`, and `aggregations` returned HTTP 200 on both. The
upstream proxy template returns HTTP 404 at `/` by design.
