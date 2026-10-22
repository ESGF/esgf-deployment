# ESGF Ansible Deployment Issue Log

**Last updated:** 2026-10-21
**Test status:** Initial Rocky Linux 8.10 deployment completed; core HTTP
endpoints returned HTTP 200.
**Scope:** Technical log and index of issues found while deploying the
`esgf-docker` Ansible playbook. This report will grow as additional operating
systems and versions are tested.

## 1. Test environment

| Item | Tested configuration |
|---|---|
| Cloud | IFCA OpenStack |
| VM | `futura-rockylinux-8-10-test` |
| OS | Rocky Linux 8.10 |
| Flavor | `cm14-compute-8`, 8 vCPU, 23 GB RAM, 30 GB disk |
| Instance ID | `13d3b364-5871-469a-b751-c86cd53c35e4` |
| Fixed / floating IP | `[REDACTED_PRIVATE_IP]` / `[REDACTED_PUBLIC_IP]` |
| Guest kernel | `4.18.0-553.el8_10.x86_64` |
| Guest Docker Engine | `26.1.3` |
| Ansible control-node version | `ansible-core 2.16.3` |
| Upstream repository | `/home/padillad/git/esgf-docker`, commit `5e2f3c3` |
| Deployment repository | FUTURA; upstream checkout kept read-only |
| Image selection | `esgfdeploy/*:latest` |
| TLS | No certificate or private key supplied; HTTP-only configuration tested |

The VM had 8 CPUs and 22,270 MiB total memory when checked from the guest.
OpenTofu reported no infrastructure drift. The Futura inventory uses strict SSH
host-key checking. No data mounts or external datasets were configured.

## 2. Issue index

| ID | Issue | Classification | Status |
|---|---|---|---|
| ANS-001 | Docker containers fail under upstream `MountFlags=shared` drop-in | Runtime/deployment bug | Futura workaround applied; upstream fix proposed |
| ANS-002 | Proxy file-permission task fails when TLS files are omitted | Configuration bug | Futura workaround applied; upstream fix proposed |
| ANS-003 | Re-running Ansible recreates every service container | Availability/idempotence issue | Reproduced; upstream fix proposed, not applied |
| ANS-004 | Duplicate `data_datasets` key in example group variables | Configuration defect | Confirmed in upstream file; no customized data-node run yet |
| ANS-005 | Futura-local playbook does not load upstream sibling `group_vars` | Test/configuration caveat | Documented; no variables staged or altered |
| ANS-006 | Initial check-mode run fails in auth uninstall before Docker exists | Check-mode limitation | Reproduced; workaround is a real deployment run |

## 3. Detailed issue entries

### ANS-001 — Docker mount namespace setting prevents container startup

- **Classification:** Runtime/deployment bug.
- **Affected system:** Rocky Linux 8.10 VM; upstream Docker role; all containers
  started on this host.
- **Symptom:** runc reported that `/usr/local/bin/tini` did not exist while
  starting the THREDDS container.
- **Evidence:** The role had created
  `/usr/lib/systemd/system/docker.service.d/mount_flags.conf` with
  `MountFlags=shared`. Inspection of the image confirmed `tini` was present.
- **Cause:** The Docker systemd mount-flags override prevented the expected
  container mount visibility on this host. The error misleadingly surfaced as
  a missing executable inside the image.
- **Fix applied in FUTURA:** The Docker role override removes the drop-in,
  reloads systemd when it is removed, and restarts Docker only in that case.
  The override also makes repository/package setup idempotent to avoid
  restarting Docker on every playbook run.
- **Upstream fix proposed:** Stop creating this drop-in, or validate whether it
  is required for supported platforms and remove it on systems where it breaks
  container startup.
- **Verification:** Containers started after removal; the drop-in was absent
  on subsequent verification. The repeat playbook run skipped the Docker
  restart task.
- **Files:** Futura [`docker override`](./ansible/overrides/roles/docker/tasks/main.yml);
  upstream reference `esgf-docker/deploy/ansible/roles/docker/tasks/main.yml`.

### ANS-002 — Proxy permissions task assumes certificate files exist

- **Classification:** Configuration bug.
- **Affected system:** Proxy role with TLS certificate and key omitted.
- **Symptom:** The play failed while attempting to set permissions on
  certificate/key paths that had not been created.
- **Cause:** Upstream conditionally installs certificate and key files but
  executes the permissions task unconditionally.
- **Fix applied in FUTURA:** Guard the permissions task so it runs only when
  both `ssl_certificate` and `ssl_private_key` are defined.
- **Upstream fix proposed:** Apply the same condition to the permission task,
  or make it verify the paths exist before changing ownership and mode.
- **Verification:** The no-certificate HTTP deployment completed, and the
  proxy started. HTTPS/certificate provisioning remains untested.
- **Files:** Futura [`proxy override`](./ansible/overrides/roles/proxy/tasks/main.yml);
  upstream reference `esgf-docker/deploy/ansible/roles/proxy/tasks/main.yml`.

### ANS-003 — Unchanged playbook rerun recreates running containers

- **Classification:** Availability/idempotence issue; not a failed deployment.
- **Affected system:** Data, index, and proxy service containers.
- **Symptom:** A second playbook run recreated all six service containers,
  interrupting service briefly even though the Docker repository and package
  tasks were unchanged.
- **Cause:** Upstream `docker_container` tasks specify `recreate: yes`.
- **Fix status:** Not changed because the upstream checkout is read-only and
  no Futura container-role override has been made for this behavior.
- **Upstream fix proposed:** Remove unconditional `recreate: yes` and let the
  Docker module recreate a container only when its image or configuration
  changes; verify desired upgrade behavior separately.
- **Verification:** Second run completed with `ok=47`, `changed=6`,
  `unreachable=0`, `failed=0`. Futura Docker repository/package setup was
  unchanged and the Docker restart task was skipped. The six service containers
  were nonetheless recreated, then the HTTP checks passed again.
- **Scope note:** This records observed rerun behavior; service interruption
  duration was not measured.

### ANS-004 — Duplicate `data_datasets` definition in upstream example

- **Classification:** Configuration defect; not yet exercised in a customized
  data-node deployment.
- **Affected file:** `esgf-docker/deploy/ansible/group_vars/data.yml`.
- **Symptom/risk:** The YAML file defines `data_datasets` twice. Standard YAML
  loading keeps the later mapping, so the earlier CMIP6/CORDEX entries are
  silently replaced by the later S3 example.
- **Fix status:** Not changed; no upstream files were edited, and the test did
  not load this file or configure data mounts.
- **Upstream fix proposed:** Combine the intended dataset examples into one
  `data_datasets` sequence, or comment out/remove the unused example mapping.
- **Verification:** Duplicate key confirmed by source inspection; runtime
  behavior with this group-vars file has not been tested in this deployment.

### ANS-005 — Local playbook does not automatically discover upstream group vars

- **Classification:** Test/configuration caveat, not an upstream deployment
  failure.
- **Symptom/risk:** The Futura playbook is stored outside the upstream
  `deploy/ansible` directory. Ansible does not automatically load the upstream
  playbook directory's sibling `group_vars` from this Futura-local playbook.
- **Fix status:** No group variables were loaded for the baseline deployment.
  This was intentional: the run used role defaults and no external datasets.
- **Recommendation:** Before testing customized datasets or index settings,
  stage reviewed variable files alongside the Futura inventory/playbook or pass
  explicit variable files, then verify the resolved variables with Ansible.
- **Verification:** Baseline playbook ran without upstream `group_vars`; data
  mount and dataset configuration were not validated.

### ANS-006 — Initial check-mode run cannot handle absent Docker daemon

- **Classification:** Check-mode validation limitation.
- **Affected system:** First dry-run against the fresh Rocky Linux 8.10 VM,
  before Docker had been installed.
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

## 4. Deployment procedure and results

Run from a host with Ansible and SSH access:

```sh
ANSIBLE_ROLES_PATH=/path/to/futura/esgf-deployment/ansible/overrides/roles:/path/to/esgf-docker/deploy/ansible/roles \
ansible-playbook \
  -i /path/to/futura/esgf-deployment/ansible/inventory.ini \
  /path/to/futura/esgf-deployment/ansible/esgf-docker-playbook.yml \
  --limit node01
```

Replace `/path/to/...` with the local repository locations. Keep the Futura
overrides first in `ANSIBLE_ROLES_PATH`; do not edit the upstream checkout.

The initial successful deployment completed with `ok=48`, `changed=13`,
`unreachable=0`, `failed=0`, and `skipped=30`. The repeat run is recorded in
ANS-003. The playbook passed Ansible syntax validation.

## 5. Service verification

The following containers were running: `proxy`, `thredds`, `fileserver`,
`solr-master`, `solr-slave`, and `search`. Through the VM floating IP, the
following checks returned HTTP 200:

- `/health`
- `/thredds/catalog/catalog.html`
- ESGF search endpoint `/esg-search/search` with a query
- Solr replication handlers for `files`, `datasets`, and `aggregations`

The root path `/` returns HTTP 404 by design in the upstream proxy template.
This is not logged as a deployment bug.

## 6. Test coverage and next entries

Only Rocky Linux 8.10 has been tested so far. Do not generalize the Docker
mount behavior or other findings to additional operating systems until they
are reproduced there.

For each new operating-system/image combination, append an issue entry with a
unique ID (`ANS-007` onward), classification, affected OS and versions,
symptom, evidence, cause, fix or workaround, verification, and upstream/FUTURA
scope. Record the image ID, flavor, upstream commit, image tags/digests,
Ansible version, and playbook recap for each test run.

Other follow-up tests: use immutable container image tags/digests, test HTTPS
and certificates, and test an intentional data-mount/dataset configuration.
