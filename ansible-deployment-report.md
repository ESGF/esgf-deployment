# ESGF Ansible Deployment Issue Log

**Last updated:** 2026-10-27
**Scope:** Tests performed and issues found while deploying the upstream
`esgf-docker` Ansible playbook.

## 1. Tests performed

| OS | Image | Environment | Docker | Ansible Core | Upstream | Configuration |
|---|---|---|---|---|---|---|
| Rocky Linux 8.10 | `RockyLinux8.10` (`fd621c1a-6a3b-44dd-9c62-2390551bc7ec`) | IFCA OpenStack, `cm14-compute-8` | 26.1.3 | 2.16.3 | `esgf-docker` `5e2f3c3` | `esgfdeploy/*:latest`; HTTP only; no datasets |
| Rocky Linux 9.8 | `RockyLinux9.8-GenericCloud-Base` (`cc0c5ae4-d593-423e-98e9-71a934c2d6bf`) | IFCA OpenStack, `cm14-compute-8` | 29.9.2 | 2.16.3 | `esgf-docker` `5e2f3c3` | `esgfdeploy/*:latest`; HTTP only; no datasets |

The Rocky Linux 9.8 test began with a clean VM and upstream roles only. The
reproducible baseline-first procedure is in the
[`Ansible test methodology`](./ansible-test-methodology.md).

## 2. OS-specific findings at a glance

The scope below identifies where a problem was observed, not every platform
on which its underlying code might be present.

| OS / scope | Findings and evidence |
|---|---|
| Rocky Linux 8.10 | The clean upstream-only run reproduced ANS-001 at THREDDS startup. Rocky 8 fixes then completed deployment; ANS-003 was reproduced on a repeat run. ANS-002 has also been observed with TLS omitted and is addressed by the Rocky 8 proxy override. ANS-006 was observed in the initial check-mode test. |
| Rocky Linux 9.8 | The upstream-only baseline reproduced ANS-007. After its pip fix, the upstream proxy reproduced ANS-002. The Rocky 9 proxy override allowed deployment. ANS-003 occurred on a repeat run. ANS-001 did not occur: all containers started with the upstream `MountFlags=shared` drop-in present. ANS-006 was not tested. |
| OS-independent configuration/test setup | ANS-004 is an upstream YAML defect confirmed by source inspection. ANS-005 concerns Ansible variable discovery from the external Futura playbook. Neither was validated in a customized deployment. |

## 3. Issue index

| ID | Issue | OS scope of observed evidence | Classification | Status |
|---|---|---|---|---|
| ANS-001 | Docker containers fail under upstream `MountFlags=shared` drop-in | Rocky Linux 8.10 reproduced on clean upstream baseline; Rocky Linux 9.8 not reproduced with drop-in present | Runtime/deployment bug | Rocky 8 workaround applied; no Rocky 9 workaround needed based on tested run |
| ANS-002 | Proxy file-permission task fails when TLS files are omitted | Reproduced on Rocky Linux 8.10 and 9.8 with no TLS | Configuration bug | OS-specific Futura workarounds applied; upstream fix proposed |
| ANS-003 | Re-running Ansible recreates every service container | Reproduced on Rocky Linux 8.10 and 9.8 | Availability/idempotence issue | Reproduced on both; upstream fix proposed, not applied |
| ANS-004 | Duplicate `data_datasets` key in example group variables | OS-independent upstream source inspection; not exercised in a deployment | Configuration defect | Confirmed in upstream file; no customized data-node run yet |
| ANS-005 | Futura-local playbook does not load upstream sibling `group_vars` | OS-independent test-setup caveat | Test/configuration caveat | Documented; no variables staged or altered |
| ANS-006 | Initial check-mode run fails in auth uninstall before Docker exists | Rocky Linux 8.10 only; Rocky Linux 9.8 not tested | Check-mode limitation | Reproduced; workaround is a real deployment run |
| ANS-007 | Rocky Linux 9 lacks pip prerequisite for Docker SDK task | Rocky Linux 9.8 only; Rocky Linux 8.10 setup passed | Platform compatibility/deployment bug | Futura workaround applied; upstream fix proposed |

## 4. Detailed issue entries

### ANS-001 — Docker mount namespace setting prevents container startup

- **Classification:** Runtime/deployment bug.
- **OS scope:** Reproduced on a clean Rocky Linux 8.10 VM using upstream roles
  only. Not reproduced on the clean Rocky Linux 9.8 VM: the unmodified
  upstream `MountFlags=shared` drop-in remained present while all six
  containers started and the HTTP checks passed.
- **Symptom:** runc reported that `/usr/local/bin/tini` did not exist while
  starting the THREDDS container.
- **Evidence:** The role had created
  `/usr/lib/systemd/system/docker.service.d/mount_flags.conf` with
  `MountFlags=shared`. Inspection of the image confirmed `tini` was present.
- **Cause:** The Docker systemd mount-flags override prevented the expected
  container mount visibility on this host. The error misleadingly surfaced as
  a missing executable inside the image.
- **Fix applied for Rocky Linux 8.10 in FUTURA:** The OS-specific Docker role
  override removes the drop-in,
  reloads systemd when it is removed, and restarts Docker only in that case.
  The override also makes repository/package setup idempotent to avoid
  restarting Docker on every playbook run.
- **Upstream fix proposed:** Remove the drop-in where it breaks container
  startup, or validate its need on supported platforms.
- **Verification:** On Rocky Linux 8.10, containers started after removal and
  the drop-in was absent. On Rocky Linux 9.8, the original drop-in was retained
  and all six containers started; this issue is therefore not recorded as
  reproduced on Rocky Linux 9.8.
- **Baseline evidence:** The clean Rocky Linux 8.10 upstream-only play stopped
  at `Start THREDDS container` with runc reporting
  `stat /usr/local/bin/tini: no such file or directory`; recap:
  `ok=19`, `changed=11`, `unreachable=0`, `failed=1`, `skipped=11`.
- **Post-fix evidence:** With the Rocky 8 override, the deployment completed
  with `ok=49`, `changed=18`, `unreachable=0`, `failed=0`, `skipped=29`.
- **Files:** Rocky 8 Futura
  [`docker override`](./ansible/overrides/rocky8/roles/docker/tasks/main.yml);
  upstream reference `esgf-docker/deploy/ansible/roles/docker/tasks/main.yml`.

### ANS-002 — Proxy permissions task assumes certificate files exist

- **Classification:** Configuration bug.
- **OS scope:** Reproduced on both Rocky Linux 8.10 and Rocky Linux 9.8 with
  certificate and key omitted. On Rocky Linux 9.8, it failed in the upstream
  proxy role after the Rocky-only pip prerequisite had been fixed.
- **Affected system:** Proxy role with TLS certificate and key omitted.
- **Symptom:** The play failed while attempting to set permissions on
  certificate/key paths that had not been created.
- **Cause:** Upstream conditionally installs certificate and key files but
  executes the permissions task unconditionally.
- **Fix applied in FUTURA:** Separate Rocky 8.10 and Rocky 9.8 proxy overrides
  guard the permissions task so it runs only when both `ssl_certificate` and
  `ssl_private_key` are defined.
- **Upstream fix proposed:** Run the permission task only when both TLS files
  are configured or present.
- **Verification:** The no-certificate HTTP deployment completed, and the
  proxy started. HTTPS/certificate provisioning remains untested.
- **Files:** Rocky 8 Futura
  [`proxy override`](./ansible/overrides/rocky8/roles/proxy/tasks/main.yml);
  Rocky 9 Futura
  [`proxy override`](./ansible/overrides/rocky9/roles/proxy/tasks/main.yml);
  upstream reference `esgf-docker/deploy/ansible/roles/proxy/tasks/main.yml`.

### ANS-003 — Unchanged playbook rerun recreates running containers

- **Classification:** Availability/idempotence issue; not a failed deployment.
- **OS scope:** Reproduced on Rocky Linux 8.10 and Rocky Linux 9.8.
- **Affected system:** Data, index, and proxy service containers.
- **Symptom:** A subsequent playbook run recreated all six service containers,
  briefly interrupting service.
- **Cause:** Upstream `docker_container` tasks specify `recreate: yes`. On
  Rocky Linux 9, the upstream Docker setup also uses command tasks that report
  changes on repeat runs and trigger a Docker restart.
- **Fix status:** Not changed. The upstream checkout is read-only, and the
  Rocky-specific Docker/proxy overrides do not change the unconditional
  container recreation behavior.
- **Upstream fix proposed:** Avoid unconditional `recreate: yes`; allow the
  Docker module to recreate containers when image or configuration changes.
- **Verification:** The Rocky Linux 8.10 second run completed with `ok=47`,
  `changed=6`, `unreachable=0`, `failed=0`. The Rocky Linux 9.8 repeat run
  completed with `ok=50`, `changed=10`, `unreachable=0`, `failed=0`; all six
  container IDs changed between runs, confirming recreation. All HTTP checks
  passed after the Rocky 9 repeat run.
- **Scope note:** Service interruption duration was not measured.

### ANS-004 — Duplicate `data_datasets` definition in upstream example

- **Classification:** Configuration defect; not yet exercised in a customized
  data-node deployment.
- **OS scope:** Not guest-OS-specific; confirmed by inspecting the upstream
  YAML source. Runtime behavior has not been tested on either Rocky release.
- **Affected file:** `esgf-docker/deploy/ansible/group_vars/data.yml`.
- **Symptom/risk:** The YAML file defines `data_datasets` twice; standard YAML
  loading can silently replace the earlier dataset mapping with the later one.
- **Fix status:** Not changed; no upstream files were edited, and the test did
  not load this file or configure data mounts.
- **Upstream fix proposed:** Combine the examples into one `data_datasets`
  sequence or remove the unused duplicate.
- **Verification:** Duplicate key confirmed by source inspection; runtime
  behavior with this group-vars file has not been tested in this deployment.

### ANS-005 — Local playbook does not automatically discover upstream group vars

- **Classification:** Test/configuration caveat, not an upstream deployment
  failure.
- **OS scope:** Not guest-OS-specific; this concerns Ansible's handling of the
  Futura-local playbook and upstream variable-file location.
- **Symptom/risk:** The Futura playbook is stored outside the upstream
  `deploy/ansible` directory. Ansible does not automatically load the upstream
  playbook directory's sibling `group_vars` from this Futura-local playbook.
- **Fix status:** No group variables were loaded for either baseline
  deployment. This was intentional: both runs used role defaults and no
  external datasets.
- **Resolution:** Supply reviewed variable files explicitly or place them where
  the Futura playbook can discover them, then verify resolved variables.
- **Verification:** Baseline playbook ran without upstream `group_vars`; data
  mount and dataset configuration were not validated.

### ANS-006 — Initial check-mode run cannot handle absent Docker daemon

- **Classification:** Check-mode validation limitation.
- **OS scope:** Reproduced during the first dry-run on Rocky Linux 8.10, before
  Docker had been installed. Not tested on Rocky Linux 9.8.
- **Affected system:** Fresh Rocky Linux 8.10 VM before Docker installation.
- **Symptom:** Ansible check mode stopped in the auth role while its uninstall
  tasks queried the Docker API, which was unavailable on the new VM.
- **Cause:** Check mode predicts changes but does not execute the preceding
  Docker installation/startup tasks. The auth uninstall tasks still invoke
  Docker container modules and require a running daemon.
- **Fix status:** No code change applied. The real deployment was run to install
  Docker before reaching those tasks.
- **Workaround:** Use check mode only after Docker is installed and running, or
  update the upstream auth role to skip Docker API operations when check mode
  is active and the Docker daemon is absent.
- **Verification:** The initial dry-run failure was observed; the subsequent
  real deployment completed successfully. A second check-mode run after Docker
  installation was not performed.

### ANS-007 — Rocky Linux 9 lacks the pip prerequisite for the Docker SDK

- **Classification:** Platform compatibility/deployment bug.
- **OS scope:** Rocky Linux 9.8 only. The Rocky Linux 8.10 deployment passed
  Docker SDK setup; this missing prerequisite has not been observed there.
- **Affected system:** Fresh Rocky Linux 9.8 host, at the Docker role's
  `Install Docker SDK` task.
- **Symptom:** The clean upstream-only Rocky Linux 9.8 deployment failed with
  `Unable to find any of pip3 to use. pip needs to be installed.` Docker CE
  packages and the upstream Docker mount drop-in had been installed, but the
  play had not installed pip.
- **Cause:** The role's pip setup covered CentOS/RHEL and AlmaLinux, but not
  Rocky Linux. On Rocky 9, the package providing pip is `python3-pip`.
- **Fix applied in FUTURA:** Added a Rocky 9-only DNF task to install
  `python3-pip` before invoking the Ansible `pip` module. The Rocky 9 override
  retains the upstream mount-flags behavior; no upstream files were changed.
- **Upstream fix proposed:** Include Rocky Linux in pip prerequisite handling
  and install the distribution-appropriate Python 3 pip package.
- **Verification:** The clean upstream-only baseline stopped at Docker SDK
  installation (`ok=8`, `changed=6`, `failed=1`, `skipped=3`). After adding
  only the Rocky pip prerequisite, the play passed Docker setup and exposed
  ANS-002 in the upstream proxy role (`ok=46`, `changed=22`, `failed=1`,
  `skipped=29`). After applying the Rocky 9 no-TLS guard, the deployment
  completed with `ok=50`, `changed=13`, `unreachable=0`, `failed=0`, and
  `skipped=30`.
- **Files:** Rocky 9 Futura
  [`docker override`](./ansible/overrides/rocky9/roles/docker/tasks/main.yml);
  upstream reference `esgf-docker/deploy/ansible/roles/docker/tasks/main.yml`.

## 5. Deployment and service verification

The clean Rocky Linux 8.10 upstream-only deployment failed at THREDDS startup
as recorded in ANS-001 (`ok=19`, `changed=11`, `unreachable=0`, `failed=1`,
`skipped=11`). After applying the Rocky 8 fixes, deployment completed with
`ok=49`, `changed=18`, `unreachable=0`, `failed=0`, `skipped=29`. The repeat
run completed with `ok=47`, `changed=6`, `unreachable=0`, `failed=0`,
`skipped=31`; all six service container IDs changed.

For Rocky Linux 9.8, the upstream-only baseline failed at Docker SDK
installation (`ok=8`, `changed=6`, `failed=1`, `skipped=3`). With only the
Rocky 9 pip fix, the run reached the upstream proxy role and failed on the
missing TLS files (`ok=46`, `changed=22`, `failed=1`, `skipped=29`). With
Rocky 9-specific fixes, deployment completed with `ok=50`, `changed=13`,
`unreachable=0`, `failed=0`, and `skipped=30`. The repeat run completed with
`ok=50`, `changed=10`, `unreachable=0`, `failed=0`, and `skipped=30`;
container IDs changed for all six services. Syntax validation passed.

The six containers (`proxy`, `thredds`, `fileserver`, `solr-master`,
`solr-slave`, and `search`) ran on both Rocky Linux 8.10 and 9.8 after their
respective fixes. The following endpoints returned HTTP 200 on both:

- `/health`
- `/thredds/catalog/catalog.html`
- ESGF search endpoint `/esg-search/search` with a query
- Solr replication handlers for `files`, `datasets`, and `aggregations`

The root path `/` returns HTTP 404 by design in the upstream proxy template;
this was not treated as a deployment issue.
