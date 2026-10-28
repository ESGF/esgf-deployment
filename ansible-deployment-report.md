# ESGF Ansible Deployment Issue Log

**Last updated:** 2026-10-27
**Purpose:** Test ESGF node deployments using the tools provided by
`esgf-docker`, across configurations and environments, and provide evidence
and proposed fixes for issues to be addressed in the upstream repository.
This report summarizes deployment-test findings; detailed investigation is
linked below.

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
| ESGF-DEP-001 | THREDDS container startup fails with upstream `MountFlags=shared` | Rocky Linux 8.10 | Rocky 8 workaround applied | [ESGF-DEP-001](./issue-reports/ESGF-DEP-001-mount-flags-container-startup.md) |
| ESGF-DEP-002 | Proxy permission task fails when TLS files are omitted | Rocky Linux 8.10 and Rocky Linux 9.8 | OS-specific workarounds applied; upstream fix proposed | [ESGF-DEP-002](./issue-reports/ESGF-DEP-002-proxy-tls-permissions.md) |
| ESGF-DEP-003 | Unchanged playbook rerun recreates service containers | Rocky Linux 8.10 and Rocky Linux 9.8 | Reproduced; upstream fix proposed, not applied | [ESGF-DEP-003](./issue-reports/ESGF-DEP-003-container-recreation.md) |
| ESGF-DEP-004 | Duplicate `data_datasets` key in upstream example variables | OS-independent (source inspection) | Confirmed by inspection; runtime behavior untested | [ESGF-DEP-004](./issue-reports/ESGF-DEP-004-duplicate-data-datasets.md) |
| ESGF-DEP-005 | Futura-local playbook does not discover upstream sibling `group_vars` | OS-independent (test setup) | Documented caveat; no group variables loaded | [ESGF-DEP-005](./issue-reports/ESGF-DEP-005-group-vars-discovery.md) |
| ESGF-DEP-006 | Initial check-mode run requires an unavailable Docker daemon | Rocky Linux 8.10 | Reproduced; no code change | [ESGF-DEP-006](./issue-reports/ESGF-DEP-006-check-mode-docker-daemon.md) |
| ESGF-DEP-007 | Rocky 9 host lacks pip prerequisite for Docker SDK | Rocky Linux 9.8 | Rocky 9 workaround applied; upstream fix proposed | [ESGF-DEP-007](./issue-reports/ESGF-DEP-007-rocky9-pip-prerequisite.md) |

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
