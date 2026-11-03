# Ansible Deployment Test Methodology

## Purpose

FUTURA's primary objective is to test ESGF node deployments using the
deployment tools provided by the upstream `esgf-docker` repository across
different configurations and environments. The work records reproducible
evidence and prepares isolated fixes for actionable findings in the FUTURA
fork of `esgf-docker`, so they can be proposed to the canonical repository.

This document defines a repeatable way to test an upstream Ansible deployment
on a new guest operating system or version. Each OS test starts with a clean
VM and unmodified upstream roles. Keep the upstream repository as the
read-only baseline; implement each actionable deployment fix on its own
branch in the FUTURA GitLab fork.

The corresponding observations and issue-by-issue results are maintained in
the [Ansible deployment test results](./ansible-deployment-test-results.md).
Deployment reports focus on `esgf-docker` behavior. Record a scenario that
does not reach Ansible as not tested; keep provisioning-tool failures in the
separate infrastructure report.

## Test rules

1. Pin and record the upstream `esgf-docker` commit used for the test. Do not
   edit the canonical upstream checkout or its `master` branch.
2. Use a fresh guest for each OS/image test. Do not treat an already-configured
   host as a clean baseline.
3. Run the initial baseline with only upstream roles in `ANSIBLE_ROLES_PATH`.
   Do not use a fix branch or local workaround in the baseline.
4. Record failures before changing anything. Apply a fix only after the issue
   has been reproduced and understood. Make a separate fork branch for each
   actionable `esgf-docker` issue; do not combine unrelated fixes.
5. Keep the upstream repository as `upstream` and the FUTURA GitLab fork as
   `origin`. Base each issue branch on the current upstream `master`, and
   push only to the fork when the user explicitly requests it.
6. Leave fix changes uncommitted and unpushed for user review. Commit or push
   only after the user explicitly asks.
7. Do not label an issue OS-specific unless its behavior is observed on that
   OS. Record “not reproduced,” “not tested,” or “not retested” explicitly.
8. Record deployment and repeat-run recaps, service state, endpoint checks,
   and unresolved limitations. A successful workaround does not prove that
   the upstream issue is absent.

Earlier tests used Futura-local OS overrides as temporary workarounds. Their
results remain historical evidence in the reports, but the temporary override
files have been removed. For post-fix verification, point
`ANSIBLE_ROLES_PATH` at the issue branch's complete role tree; a role earlier
in that path shadows the upstream role as a whole. Keep configuration-only
adaptations (such as pointing a template path at the upstream checkout)
distinct from code fixes in reports.

## 1. Prepare and record the test inputs

Before provisioning, record:

- OS name/version, source image name/UUID and checksum where available.
- Cloud/project, flavor, VM name, network, security group, and quotas.
- OpenTofu and provider versions; whether a shared or per-test state is used.
- Upstream repository commit and working-tree status.
- Ansible control-node version and the inventory/playbook paths.
- Deployment configuration, including image tags, TLS choice, and datasets.

Keep credentials out of the report and source control. Confirm that the guest
image's minimum resource requirements fit the selected flavor.
Keep previously tested image IDs as commented options next to the active
`image_id` default in OpenTofu `main.tf`; change only the active selection
when switching test OSes.

## 2. Provision a clean VM

Use the existing OpenTofu workflow and review the complete plan before
applying. For a replacement in the current single-VM state, a targeted plan
can be generated with:

```sh
tofu plan -replace=openstack_compute_instance_v2.vm -out=clean-vm.tfplan
tofu show clean-vm.tfplan
```

Apply only after confirming the plan replaces the intended VM and does not
destroy unrelated resources:

```sh
tofu apply clean-vm.tfplan
```

If OpenStack reports that an old port is still in use, inspect the exact port
and its device association first. Clear a stale device association only after
confirming the old server is gone. If Nova still refuses the port, review a
targeted port-replacement plan and verify the floating IP will be associated
with the new port. Never destroy the whole state merely to obtain a clean
guest. Record any fixed-IP or port-ID change.

For this recovery case, the checks and narrowly scoped cleanup are:

```sh
openstack server show <old-server-id>
openstack port show <old-port-id> -f yaml
openstack port unset --device --device-owner <old-port-id>
openstack port show <old-port-id> -f yaml
```

Only clear the association when the port is confirmed to reference the
deleted server. If scheduling still reports the port in use, inspect a
targeted `tofu plan -replace=openstack_networking_port_v2.vm` before applying;
confirm the existing floating IP remains managed and will be moved to the new
port.

Wait for cloud-init and SSH to finish. Verify the new SSH host-key fingerprint
against the VM console before updating the pinned key in the Futura inventory.
For example, compare the console fingerprint with:

```sh
ssh-keyscan -t ed25519 <floating-ip> 2>/dev/null | ssh-keygen -lf -
```

Then check Ansible connectivity and collect guest facts before deployment:

```sh
ansible node01 -i "$FUTURA_ANSIBLE/inventory.ini" -m ping
ansible node01 -i "$FUTURA_ANSIBLE/inventory.ini" -m setup
```

Record the instance ID, fixed/floating IPs, kernel, memory, and root
filesystem. Confirm the guest OS from `/etc/os-release` or its release file;
do not rely on the image's display name alone.

## 3. Run the upstream-only baseline

Set paths to the pinned upstream role directory and the Futura playbook:

```sh
UPSTREAM_ROLES=/path/to/esgf-docker/deploy/ansible/roles
FUTURA_ANSIBLE=/path/to/futura/esgf-deployment/ansible
```

For the data-serving scenario, keep the test's single `data_datasets` entry
and `data_mounts` in
`$FUTURA_ANSIBLE/group_vars/data.yml`, which is discoverable relative to the
inventory. Confirm the resolved variables before deployment:

```sh
ansible-inventory -i "$FUTURA_ANSIBLE/inventory.ini" --host node01
```

The tracked test fixture is a small synthetic CF-1.6 NetCDF file. Prepare it
on the clean host before the upstream-only deployment; this playbook only
creates the test data and does not apply deployment roles:

```sh
ansible-playbook -i "$FUTURA_ANSIBLE/inventory.ini" \
  "$FUTURA_ANSIBLE/prepare-test-data.yml"
```

The source CDL can regenerate the binary fixture with
`ncgen -k classic -o test-data/futura-test.nc test-data/futura-test.cdl`.
Regeneration requires `ncgen` (provided by the `netcdf-bin` package on Debian-
based systems); the checked-in NetCDF file can be used without regenerating it.
Do not load the upstream example `group_vars/data.yml` for this scenario: it
contains the duplicate key tracked in
[ESGF-DEP-004](./issue-reports/ESGF-DEP-004-duplicate-data-datasets.md).

First validate syntax using upstream roles only:

```sh
ANSIBLE_ROLES_PATH="$UPSTREAM_ROLES" \
  ansible-playbook -i "$FUTURA_ANSIBLE/inventory.ini" \
  "$FUTURA_ANSIBLE/esgf-docker-playbook.yml" --syntax-check
```

Run the real deployment with the same upstream-only role path:

```sh
ANSIBLE_ROLES_PATH="$UPSTREAM_ROLES" \
  ansible-playbook -i "$FUTURA_ANSIBLE/inventory.ini" \
  "$FUTURA_ANSIBLE/esgf-docker-playbook.yml" --limit node01
```

Do not add any Futura workaround or fork fix for this run. If the play fails,
preserve the relevant task, error, and recap as baseline evidence. Inspect
guest packages, service state, and logs only as needed to establish the cause.
Classify each observation as guest-OS/platform-specific, configuration-related,
upstream behavior shared across platforms, or test/setup limitation.

If check-mode behavior is part of the test scope, run and record it separately
on the fresh host before the real deployment. It does not replace the real
upstream-only baseline, because check mode does not execute installation
tasks.

## 4. Create and verify an issue-specific fork branch

For each actionable `esgf-docker` issue, create one branch in the FUTURA fork.
Use remotes named `origin` (the fork) and `upstream` (the canonical GitHub
repository). Fetch both, base the branch on the current upstream `master`,
and use a descriptive issue-specific name:

```sh
git fetch upstream
git fetch origin
git switch -c fix/ESGF-DEP-###-short-description upstream/master
```

Change only what is needed to fix that finding. Keep OS-specific conditions
in the shared upstream role when evidence shows the fix should be conditional
on the OS. Leave the changes uncommitted and unpushed so the user can review
them first. Do not commit or push unless explicitly requested. After approval,
the following commands can be used:

```sh
git add <changed-files>
git diff --cached --check
git commit -m "Fix <short issue description>"
git push -u origin fix/ESGF-DEP-###-short-description
```

Do not push fix branches to `upstream`. Do not combine unrelated issue fixes.
If a finding is a FUTURA test-setup caveat rather than an `esgf-docker` code
issue, document it and do not create an irrelevant upstream fix branch.

Validate syntax and rerun using the role tree from the issue branch:

```sh
ESGF_DOCKER_FORK=/path/to/esgf-docker
ANSIBLE_ROLES_PATH="$ESGF_DOCKER_FORK/deploy/ansible/roles" \
  ansible-playbook -i "$FUTURA_ANSIBLE/inventory.ini" \
  "$FUTURA_ANSIBLE/esgf-docker-playbook.yml" --syntax-check

ANSIBLE_ROLES_PATH="$ESGF_DOCKER_FORK/deploy/ansible/roles" \
  ansible-playbook -i "$FUTURA_ANSIBLE/inventory.ini" \
  "$FUTURA_ANSIBLE/esgf-docker-playbook.yml" --limit node01
```

Record the branch name, base upstream commit, working-tree changes, syntax/test
results, and any OS/configuration limits in the issue report. Record a fix
commit only after the user requests and approves a commit. Do not describe a
baseline failure as fixed until the branch's change has been exercised and
the corresponding behavior verified. Use a new issue branch for a separate
finding exposed during a retry.

## 5. Verify the deployment

After a successful run, record the full Ansible recap and check the guest
service state and expected containers. For the current HTTP-only test:

```sh
ansible node01 -i "$FUTURA_ANSIBLE/inventory.ini" -b \
  -m shell -a 'systemctl is-active docker; docker ps'
```

For HTTP status checks, use the current floating IP:

```sh
curl --connect-timeout 10 --max-time 30 -sS -o /dev/null \
  -w '%{http_code}\n' "http://<floating-ip>/health"
```

From a host with access to the test VM, check each configured endpoint:

```text
/health
/thredds/catalog/catalog.html
/esg-search/search?query=*&limit=1
/solr/files/replication?command=details
/solr/datasets/replication?command=details
/solr/aggregations/replication?command=details
```

Record HTTP status for each. Do not count an endpoint as verified based only
on the Ansible task succeeding. Also record whether TLS, external datasets,
and any optional service integrations were actually configured and tested.

For the synthetic data-serving scenario, also verify the dataset-specific
THREDDS catalog, the `fileServer` URL, and an OPeNDAP metadata/data request.
Download the served file and compare its checksum with the local fixture;
inspect its NetCDF header/data with `ncdump`. Confirm the host bind mount is
present in both THREDDS and fileserver containers, read-only, and readable.
Treat a healthy ESGF search response separately from indexing the synthetic
dataset: do not claim indexing unless the dataset is registered and appears
in the search results.

## 6. Test repeat-run behavior

Once the deployment is complete, capture container IDs and status, run the
same playbook again with the same OS-specific role path, then compare IDs and
the second recap. This tests idempotence separately from first-run
installation. Perform this only on the disposable test VM because a run may
restart Docker or recreate containers and briefly interrupt service.

Record whether containers were recreated, whether Docker restarted, and
whether endpoint checks recovered. Do not infer repeat-run behavior from the
first successful deployment.

## 7. Update all relevant reports

After every new OS, image, distribution, or deployment-environment test,
update all reports affected by its results before considering the test
complete:

- **Ansible deployment report:** Keep the environment matrix and verification
  results current. Add a concise finding row for each issue, state only the
  distributions where it was observed in the observed-scope column, and
  distinguish OS-independent findings from distribution-specific ones.
- **Issue reports:** Create one detailed report under
  `issue-reports/` for every new issue, linked from the general report.
  Update existing issue reports when new evidence changes their observed
  scope, status, proposed fix, or verification. Include baseline evidence,
  retry results, relevant recaps, limits, and the fork branch/commit when a
  fix exists. Keep the general report concise; do not duplicate the detailed
  diagnosis there.
- **OpenTofu report:** Add each provisioned test environment, including OS,
  image name/UUID, flavor, and provisioning outcome. Report tested
  environments rather than which VM happens to be active. Keep infrastructure
  provisioning details in this separate report, not in the ESGF deployment
  results. Keep previous tested image IDs as commented options in `main.tf`.
- **Methodology:** Update the procedure if the test reveals a reusable
  improvement, a new prerequisite, or a recovery step that should be applied
  in future OS tests.

For each test, record its date, pinned upstream commit, clean upstream-only
baseline, each OS-specific retry and full playbook recap, repeat-run result,
service/container and endpoint verification, and anything not reproduced or
not tested. Assign each newly observed deployment finding a unique
`ESGF-DEP-###` ID. This prefix identifies findings from ESGF deployment
testing; it does not imply every entry is a confirmed upstream bug. Keep
each code fix in a dedicated fork branch named for its finding. Never describe
a baseline failure as fixed until the corresponding behavior is verified.

## Rocky Linux 9.8 data-serving test performed on 2026-10-30

The clean Rocky Linux 9.8 data-serving test used upstream roles first. The
baseline failed at Docker SDK installation because the pip prerequisite was
missing (`ok=8`, `changed=6`, `failed=1`, `skipped=3`). At the time of this
test, the existing Rocky 9 local overrides supplied the pip prerequisite and
HTTP-only proxy guard; deployment passed (`ok=50`, `changed=26`,
`unreachable=0`, `failed=0`, `skipped=30`). Under the new workflow, fixes for
actionable findings are developed and verified on their individual fork
branches. The fixture mount, THREDDS and fileserver access, file checksum,
OPeNDAP, service endpoints, and repeat-run behavior passed verification. The
repeat recreated all six service containers (`ok=50`, `changed=10`,
`failed=0`, `skipped=30`). Detailed evidence is in the [deployment test
results](./ansible-deployment-test-results.md#rocky-linux-98-data-serving-scenario--2026-10-30).
