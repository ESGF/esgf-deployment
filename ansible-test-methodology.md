# Ansible deployment test methodology

## Purpose

FUTURA's primary objective is to test ESGF node deployments using the
deployment tools provided by the upstream `esgf-docker` repository across
different configurations and environments. The work records reproducible
evidence and proposes fixes for issues so they can be addressed in
`esgf-docker`; it does not modify that upstream checkout.

This document defines a repeatable way to test an upstream Ansible deployment
on a new guest operating system or version. Each OS test starts with a clean
VM and unmodified upstream roles. Fixes discovered on one OS must not leak
into another OS's baseline or override tree.

The corresponding observations and issue-by-issue results are maintained in
the [Ansible deployment issue log](./ansible-deployment-report.md).

## Test rules

1. Pin and record the upstream `esgf-docker` commit used for the test. Do not
   edit the upstream checkout.
2. Use a fresh guest for each OS/image test. Do not treat an already-configured
   host as a clean baseline.
3. Run the initial baseline with only upstream roles in `ANSIBLE_ROLES_PATH`.
   Do not use role overrides or fixes from a previously tested OS.
4. Record failures before changing anything. Apply a fix only after the issue
   has been reproduced and understood.
5. Keep each OS's fixes in a separate tree, for example:
   `ansible/overrides/rocky8/roles/` and `ansible/overrides/rocky9/roles/`.
   Put the tested OS's override path before upstream roles only for that OS's
   post-baseline runs.
6. Do not label an issue OS-specific unless its behavior is observed on that
   OS. Record “not reproduced,” “not tested,” or “not retested” explicitly.
7. Record deployment and repeat-run recaps, service state, endpoint checks,
   and unresolved limitations. A successful workaround does not prove that
   the upstream issue is absent.

An Ansible role found earlier in `ANSIBLE_ROLES_PATH` shadows the upstream
role as a whole; role files are not overlaid file-by-file. When overriding a
role, base its files on the same pinned upstream commit, preserve the files
needed by that role, and make only the OS-test-specific changes. Keep
configuration-only adaptations (such as pointing a template path at the
upstream checkout) distinct from bug fixes in the report.

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

Do not add any Futura OS override for this run. If the play fails, preserve
the relevant task, error, and recap as baseline evidence. Inspect guest
packages, service state, and logs only as needed to establish the cause.
Classify each observation as guest-OS/platform-specific, configuration-related,
upstream behavior shared across platforms, or test/setup limitation.

If check-mode behavior is part of the test scope, run and record it separately
on the fresh host before the real deployment. It does not replace the real
upstream-only baseline, because check mode does not execute installation
tasks.

## 4. Apply fixes in the tested OS tree only

Create or update only the tree for the guest under test, for example:

```text
ansible/overrides/rocky8/roles/
ansible/overrides/rocky9/roles/
```

For a role override, start from the pinned upstream role and change only the
tasks/defaults necessary to address an issue reproduced during that OS test.
Do not copy fixes from another OS merely because they were needed there.
If the same issue is independently reproduced on multiple OSs, keep the
overrides isolated unless a later deliberate refactor is requested.

Validate syntax and rerun with only the current OS's override tree before the
upstream role path:

```sh
OS_OVERRIDE_ROLES="$FUTURA_ANSIBLE/overrides/<os-version>/roles"
ANSIBLE_ROLES_PATH="$OS_OVERRIDE_ROLES:$UPSTREAM_ROLES" \
  ansible-playbook -i "$FUTURA_ANSIBLE/inventory.ini" \
  "$FUTURA_ANSIBLE/esgf-docker-playbook.yml" --syntax-check

ANSIBLE_ROLES_PATH="$OS_OVERRIDE_ROLES:$UPSTREAM_ROLES" \
  ansible-playbook -i "$FUTURA_ANSIBLE/inventory.ini" \
  "$FUTURA_ANSIBLE/esgf-docker-playbook.yml" --limit node01
```

If a retry exposes another failure, record it against that OS, then add the
next minimal fix to the same OS tree and rerun. Do not describe a baseline
failure as fixed until the retry completes and the corresponding behavior is
verified.

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
  retry results, relevant recaps, limits, and the OS-specific override or
  upstream reference. Keep the general report concise; do not duplicate the
  detailed diagnosis there.
- **OpenTofu report:** Add each provisioned test environment, including OS,
  image name/UUID, flavor, and provisioning outcome. Report tested
  environments rather than which VM happens to be active. Keep previous
  tested image IDs as commented options in `main.tf`.
- **Methodology:** Update the procedure if the test reveals a reusable
  improvement, a new prerequisite, or a recovery step that should be applied
  in future OS tests.

For each test, record its date, pinned upstream commit, clean upstream-only
baseline, each OS-specific retry and full playbook recap, repeat-run result,
service/container and endpoint verification, and anything not reproduced or
not tested. Assign each newly observed deployment finding a unique
`ESGF-DEP-###` ID. This prefix identifies findings from ESGF deployment
testing; it does not imply every entry is a confirmed upstream bug. Keep
OS-specific fixes in their own override trees, and never describe a baseline
failure as fixed until the corresponding behavior is verified.

## Rocky Linux 9.8 clean retest performed on 2026-10-22

The clean test followed the upstream-only baseline sequence:

1. Replaced the VM and stale network port while retaining the floating IP.
   The new instance is `211d2ba9-bba2-44d7-a041-c83eda83228a`; the fixed IP
   is `TESTBED_PRIVATE_IP`.
2. Verified Rocky Linux 9.8 and the new SSH host key against cloud-init
   console output.
3. Ran syntax validation and deployment with upstream roles only. It failed
   at Docker SDK installation because Rocky's `python3-pip` prerequisite was
   not installed (`ok=8`, `changed=6`, `failed=1`, `skipped=3`).
4. Added only the pip prerequisite to the Rocky 9 Docker override and reran.
   Deployment reached the upstream proxy role, where the no-TLS file
   permissions task failed (`ok=46`, `changed=22`, `failed=1`,
   `skipped=29`). The five non-proxy containers started with the upstream
   `MountFlags=shared` drop-in present.
5. Added the no-TLS condition to the Rocky 9 proxy override. The full run
   completed (`ok=50`, `changed=13`, `unreachable=0`, `failed=0`,
   `skipped=30`); all six endpoints returned HTTP 200.
6. Repeated the playbook; all six service container IDs changed
   (`ok=50`, `changed=10`, `unreachable=0`, `failed=0`, `skipped=30`).
   All endpoints returned HTTP 200 after the repeat run.

The `MountFlags=shared` container-start failure was not reproduced on this
Rocky 9.8 guest. The clean-host check-mode behavior was not tested during this
retest.
