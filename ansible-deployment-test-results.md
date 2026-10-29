# ESGF Ansible Deployment Test Results

**Last updated:** 2026-10-29
**Purpose:** Test ESGF node deployments using the tools provided by
`esgf-docker`, across configurations and environments, and provide evidence
and proposed fixes for issues to be addressed in the upstream repository.
This report summarizes deployment-test findings; detailed investigation is
linked below.
It focuses on `esgf-docker` deployment behavior. Scenarios without deployment
evidence are marked as not tested; provisioning-tool failures are outside
this report's scope.

## 1. Tested environments

| OS | Image | Environment | Docker | Ansible Core | Upstream | Configuration |
|---|---|---|---|---|---|---|
| Rocky Linux 8.10 | `RockyLinux8.10` (`fd621c1a-6a3b-44dd-9c62-2390551bc7ec`) | IFCA OpenStack, `cm14-compute-8` | 26.1.3 | 2.16.3 | `esgf-docker` `5e2f3c3` | One synthetic CF-1.6 NetCDF via read-only host mount; HTTP only; not registered in ESGF search |
| Rocky Linux 9.8 | `RockyLinux9.8-GenericCloud-Base` (`cc0c5ae4-d593-423e-98e9-71a934c2d6bf`) | IFCA OpenStack, `cm14-compute-8` | — | — | — | Data-serving deployment not tested |

The Rocky Linux 8.10 test began with a clean host and upstream roles. The
Rocky Linux 9.8 data-serving scenario has not been tested. The reproducible
baseline-first procedure is in the
[`Ansible deployment test methodology`](./ansible-deployment-test-methodology.md).

## 2. Findings

The OS scope below describes where each finding was observed; it does not
establish that a finding cannot occur on other systems.

| ID | Finding | Observed scope | Status | Detailed report |
|---|---|---|---|---|
| ESGF-DEP-001 | THREDDS container startup fails with upstream `MountFlags=shared` | Rocky Linux 8.10 | Rocky 8 workaround applied | [ESGF-DEP-001](./issue-reports/ESGF-DEP-001-mount-flags-container-startup.md) |
| ESGF-DEP-002 | Proxy permission task fails when TLS files are omitted | Rocky Linux 8.10 and Rocky Linux 9.8 | OS-specific workarounds applied; upstream fix proposed | [ESGF-DEP-002](./issue-reports/ESGF-DEP-002-proxy-tls-permissions.md) |
| ESGF-DEP-003 | Unchanged playbook rerun recreates service containers | Rocky Linux 8.10 and Rocky Linux 9.8 | Reproduced; upstream fix proposed, not applied | [ESGF-DEP-003](./issue-reports/ESGF-DEP-003-container-recreation.md) |
| ESGF-DEP-004 | Duplicate `data_datasets` key in upstream example variables | OS-independent (source inspection) | Confirmed by inspection; runtime behavior untested | [ESGF-DEP-004](./issue-reports/ESGF-DEP-004-duplicate-data-datasets.md) |
| ESGF-DEP-005 | Futura-local playbook does not discover upstream sibling `group_vars` | OS-independent (test setup) | Inventory-local group-vars workaround verified | [ESGF-DEP-005](./issue-reports/ESGF-DEP-005-group-vars-discovery.md) |
| ESGF-DEP-006 | Initial check-mode run requires an unavailable Docker daemon | Rocky Linux 8.10 | Reproduced; no code change | [ESGF-DEP-006](./issue-reports/ESGF-DEP-006-check-mode-docker-daemon.md) |
| ESGF-DEP-007 | Rocky 9 host lacks pip prerequisite for Docker SDK | Rocky Linux 9.8 | Rocky 9 workaround applied; upstream fix proposed | [ESGF-DEP-007](./issue-reports/ESGF-DEP-007-rocky9-pip-prerequisite.md) |

Rocky Linux 9.8 scope in the existing issue reports reflects earlier
deployment observations; those findings were not re-tested as part of the
data-serving scenario.

## 3. Deployment verification

| OS | Clean upstream-only baseline | With OS-specific fixes | Repeat run |
|---|---|---|---|
| Rocky Linux 8.10 | Failed at THREDDS startup: `ok=19`, `changed=11`, `failed=1`, `skipped=11` | Passed: `ok=49`, `changed=18`, `failed=0`, `skipped=29` | Passed: `ok=47`, `changed=6`, `failed=0`, `skipped=31`; all six service container IDs changed |
| Rocky Linux 9.8 | Not tested for data serving | Not run | Not run |

After applying Rocky 8-specific fixes, `proxy`, `thredds`, `fileserver`,
`solr-master`, `solr-slave`, and `search` ran on Rocky Linux 8.10. The
`/health`, THREDDS catalog, ESGF search query, and Solr replication endpoints
for `files`, `datasets`, and `aggregations` returned HTTP 200. The upstream
proxy template returns HTTP 404 at `/` by design. Rocky Linux 9.8 service and
data-serving behavior has not been tested in this scenario.

### Rocky Linux 8.10 data-serving scenario — 2026-10-29

The clean VM used the Rocky Linux 8.10 image and `cm14-compute-8` flavor.
Futura inventory group variables configured one host mount,
`/srv/futura-esgf-test-data` to `/data`, and one dataset named `FUTURA
synthetic dataset` at `/data/futura-test`, exposed as `futura_test`. The
fixture is a 720-byte synthetic CF-1.6 NetCDF file with `tas` values 280 K
and 281 K. Its SHA-256 is
`687be433b5375071a25866dd7c66697813b9eb7153cfd9afe3515d18b14c8720`.
`ansible-inventory --host node01` confirmed the single mount and dataset
variables were loaded. Fixture preparation passed (`ok=3`, `changed=2`).

The upstream-only clean baseline reproduced
[ESGF-DEP-001](./issue-reports/ESGF-DEP-001-mount-flags-container-startup.md)
at THREDDS startup with runc reporting that `/usr/local/bin/tini` was absent
(`ok=19`, `changed=11`, `unreachable=0`, `failed=1`, `skipped=11`). The
rendered THREDDS catalog contained the configured dataset scan, but deployment
stopped before the containers could serve it.

With the existing Rocky 8-specific overrides, deployment completed
(`ok=49`, `changed=18`, `unreachable=0`, `failed=0`, `skipped=29`). The
`/data` bind mount was present and read-only in both THREDDS and fileserver;
the fixture was readable in both containers. The root and dataset-specific
THREDDS catalogs, fileServer request, OPeNDAP `.dds` and `.dods?tas` requests,
health endpoint, ESGF search endpoint, and all three Solr replication
endpoints returned HTTP 200. Downloading the fileServer URL produced a file
with the same SHA-256 as the local fixture; `ncdump` confirmed the NetCDF
header and `tas` values.

The search endpoint returned a valid response with `numFound=0`. The
synthetic dataset was not registered in ESGF search, so this run verifies
search service availability, not dataset indexing or federation publication.

The fixture-preparation rerun was idempotent (`ok=3`, `changed=0`). The
Ansible repeat run passed (`ok=47`, `changed=6`, `unreachable=0`, `failed=0`,
`skipped=31`); all six service container IDs changed, consistent with
[ESGF-DEP-003](./issue-reports/ESGF-DEP-003-container-recreation.md).
All eight endpoint checks, including the dataset catalog and fileServer,
returned HTTP 200 after the repeat.

### Rocky Linux 9.8 data-serving scenario

The Rocky Linux 9.8 data-serving deployment has not been tested. No Ansible
baseline, OS-specific deployment run, fixture verification, endpoint checks,
or repeat-run results are available for this scenario.
