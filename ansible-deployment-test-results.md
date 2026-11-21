# ESGF Ansible Deployment Test Results

**Last updated:** 2026-11-21
**Purpose:** Test ESGF data-node deployments using the tools provided by
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
| Rocky Linux 8.10 | `RockyLinux8.10` (`fd621c1a-6a3b-44dd-9c62-2390551bc7ec`) | IFCA OpenStack, `cm14-compute-8` | 26.1.3 | 2.16.3 | `esgf-docker` `5e2f3c3` | One synthetic CF-1.6 NetCDF via read-only host mount; HTTP only |
| Rocky Linux 9.8 | `RockyLinux9.8-GenericCloud-Base` (`cc0c5ae4-d593-423e-98e9-71a934c2d6bf`) | IFCA OpenStack, `cm14-compute-8` | 29.9.2 | 2.16.3 | `esgf-docker` `5e2f3c3` | One synthetic CF-1.6 NetCDF via read-only host mount; HTTP only |

Both data-serving tests began with clean hosts and upstream roles. The
reproducible baseline-first procedure is in the
[`Ansible deployment test methodology`](./ansible-deployment-test-methodology.md).
The original test inventory also enabled additional non-data services. The
results below retain only the data-serving findings; the full deployment
recaps are historical and are not data-node-only baselines. Repeat the tests
with the current data-only inventory to establish data-node-only baselines.

## 2. Findings

The OS scope below describes where each finding was observed; it does not
establish that a finding cannot occur on other systems.

| ID | Finding | Observed scope | Status | Detailed report |
|---|---|---|---|---|
| ESGF-DEP-001 | THREDDS container startup fails with upstream `MountFlags=shared` | Rocky Linux 8.10 | Fix committed to fork branch; full branch deployment retest pending | [ESGF-DEP-001](./issue-reports/ESGF-DEP-001-mount-flags-container-startup.md) |
| ESGF-DEP-002 | Proxy permission task fails when TLS files are omitted | Rocky Linux 8.10 and Rocky Linux 9.8 | Fork commit `32c6b13`; syntax check passes; deployment verification pending | [ESGF-DEP-002](./issue-reports/ESGF-DEP-002-proxy-tls-permissions.md) |
| ESGF-DEP-003 | Unchanged playbook rerun recreates service containers | Rocky Linux 8.10 and Rocky Linux 9.8 | Fork commit `69dc037`; syntax check passes; deployment verification pending | [ESGF-DEP-003](./issue-reports/ESGF-DEP-003-container-recreation.md) |
| ESGF-DEP-004 | Duplicate `data_datasets` key in upstream example variables | OS-independent (source inspection) | Fork commit `d2fea8f`; syntax and variable resolution pass; full deployment pending | [ESGF-DEP-004](./issue-reports/ESGF-DEP-004-duplicate-data-datasets.md) |
| ESGF-DEP-005 | Futura-local playbook does not discover upstream sibling `group_vars` | OS-independent (test setup) | Inventory-local group-vars configuration verified; no upstream fix applicable | [ESGF-DEP-005](./issue-reports/ESGF-DEP-005-group-vars-discovery.md) |
| ESGF-DEP-006 | Initial check-mode run requires an unavailable Docker daemon | Rocky Linux 8.10 | Retained as a check-mode limitation; no code fix branch | [ESGF-DEP-006](./issue-reports/ESGF-DEP-006-check-mode-docker-daemon.md) |
| ESGF-DEP-007 | Rocky 9 host lacks pip prerequisite for Docker SDK | Rocky Linux 9.8 | Fork commit `4ae60b8`; syntax check passes; deployment verification pending | [ESGF-DEP-007](./issue-reports/ESGF-DEP-007-rocky9-pip-prerequisite.md) |

## 3. Deployment verification

| OS | Clean upstream-only baseline | With historical temporary FUTURA overrides (now removed) | Repeat run |
|---|---|---|---|
| Rocky Linux 8.10 | Failed at THREDDS startup: `ok=19`, `changed=11`, `failed=1`, `skipped=11` | Passed: `ok=49`, `changed=18`, `failed=0`, `skipped=29` | Passed: `ok=47`, `changed=6`, `failed=0`, `skipped=31`; all configured container IDs changed |
| Rocky Linux 9.8 | Failed at Docker SDK installation: `ok=8`, `changed=6`, `failed=1`, `skipped=3` | Passed: `ok=50`, `changed=26`, `failed=0`, `skipped=30` | Passed: `ok=50`, `changed=10`, `failed=0`, `skipped=30`; all configured container IDs changed |

After applying the historical local workarounds, the proxy, THREDDS, and
fileserver containers ran on both operating systems. The health endpoint,
THREDDS catalogs, fileServer, and OPeNDAP data endpoints returned HTTP 200.
The upstream proxy template returns HTTP 404 at `/` by design.

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

With the Rocky 8-specific local test override used at the time, deployment
completed (`ok=49`, `changed=18`, `unreachable=0`, `failed=0`, `skipped=29`).
The `/data` bind mount was present and read-only in both THREDDS and
fileserver; the fixture was readable in both containers. The root and
dataset-specific THREDDS catalogs, fileServer request, OPeNDAP `.dds` and
`.dods?tas` requests, and health endpoint returned HTTP 200. Downloading the
fileServer URL produced a file with the same SHA-256 as the local fixture;
`ncdump` confirmed the NetCDF header and `tas` values.

The fixture-preparation rerun was idempotent (`ok=3`, `changed=0`). The
Ansible repeat run passed (`ok=47`, `changed=6`, `unreachable=0`, `failed=0`,
`skipped=31`); all configured container IDs changed, consistent with
[ESGF-DEP-003](./issue-reports/ESGF-DEP-003-container-recreation.md).
All data-serving endpoint checks, including the dataset catalog and
fileServer, returned HTTP 200 after the repeat.

### Rocky Linux 9.8 data-serving scenario — 2026-10-30

The clean VM used `RockyLinux9.8-GenericCloud-Base` and flavor
`cm14-compute-8`. The guest reported Rocky Linux 9.8, kernel
`5.14.0-687.10.1.el9_8.0.1.x86_64`, and Docker 29.9.2. The test used Ansible
Core 2.16.3 and upstream `esgf-docker` commit `5e2f3c3`. Inventory resolution
confirmed the single `data_datasets` entry and `/srv/futura-esgf-test-data`
to `/data` host mount. Fixture preparation passed (`ok=3`, `changed=2`).

The clean upstream-only deployment passed syntax validation but failed at
`Install Docker SDK`: Rocky's upstream prerequisite tasks did not install
pip, so Ansible reported `Unable to find any of pip3 to use. pip needs to be
installed.` The baseline recap was `ok=8`, `changed=6`, `unreachable=0`,
`failed=1`, `skipped=3`, reproducing
[ESGF-DEP-007](./issue-reports/ESGF-DEP-007-rocky9-pip-prerequisite.md).

With the Rocky 9 local test overrides used at the time, deployment completed
(`ok=50`, `changed=26`, `unreachable=0`, `failed=0`, `skipped=30`). The data
service containers ran. The `/data` bind mount was read-only in both
THREDDS and fileserver, and the fixture was readable in both containers. The
root and dataset THREDDS catalogs, fileServer, OPeNDAP `.dds` and `.dods?tas`,
and health endpoints all returned HTTP 200. The fileServer download matched the
fixture SHA-256
`687be433b5375071a25866dd7c66697813b9eb7153cfd9afe3515d18b14c8720`;
`ncdump` confirmed `tas` values 280 K and 281 K.

Fixture preparation was idempotent on rerun (`ok=3`, `changed=0`). The
unchanged deployment repeat passed (`ok=50`, `changed=10`, `unreachable=0`,
`failed=0`, `skipped=30`), but all configured container IDs changed,
reproducing [ESGF-DEP-003](./issue-reports/ESGF-DEP-003-container-recreation.md).
All data-serving endpoint checks listed above returned HTTP 200 after the repeat.
