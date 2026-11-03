# ESGF-DEP-001 — Docker mount setting prevents container startup

## Summary

- **Classification:** Runtime/deployment issue.
- **Observed scope:** Reproduced on a clean Rocky Linux 8.10 host; not
  reproduced on Rocky Linux 9.8.
- **Status:** Dedicated fix branch pushed to the FUTURA `esgf-docker` fork.
  Full deployment verification using that branch is pending. No Rocky 9
  workaround was needed in the tested run.

## Evidence

On the Rocky Linux 8.10 upstream-only baseline, the play failed at `Start
THREDDS container`. runc reported:

```text
stat /usr/local/bin/tini: no such file or directory
```

The Docker role had installed
`/usr/lib/systemd/system/docker.service.d/mount_flags.conf` with
`MountFlags=shared`. Inspection confirmed that `tini` existed in the container
image. The baseline recap was `ok=19`, `changed=11`, `unreachable=0`,
`failed=1`, `skipped=11`.

The test on Rocky Linux 9.8 did not reproduce the failure: the upstream
mount-flags drop-in remained present, all six containers started, and HTTP
checks passed.

## Assessment

The host's Docker mount configuration is the leading explanation for the
Rocky 8 failure, with the missing-executable message likely reflecting an
unexpected container mount view rather than a missing file in the image.
The tests do not isolate the precise kernel/runtime interaction, so this
cause should be treated as evidence-based but not conclusively proven.

The base image Dockerfile in the tested `esgf-docker` revision starts from
`rockylinux:9`, including when the VM host is Rocky Linux 8. Container
userspace and host OS are distinct; containers use the host kernel. No
Rocky 8-based container image was tested, so these results do not establish
whether changing the container base would affect the failure. The playbook
used mutable `esgfdeploy/*:latest` tags, and the exact image digests used in
each run were not recorded.

## Fix branch and proposed upstream change

The historical Rocky 8 local test override removed the drop-in, reloaded
systemd when it was removed, and restarted Docker only in that case. That
override was used for the successful Rocky 8 deployment verification below.

The issue-specific fix is committed on
[`fix/ESGF-DEP-001-rocky8-mount-flags`](https://gitlab.com/sci-uc/scds/futura/esgf-docker/-/tree/fix/ESGF-DEP-001-rocky8-mount-flags)
at `b5a2c57`. It skips creating the `MountFlags=shared` drop-in on Rocky
Linux 8, removes an existing copy, and reloads systemd when its removal
changes the unit configuration. Other distributions retain the upstream
behavior.

The branch passes Ansible syntax validation. A full Rocky 8 deployment using
the branch has not yet been run; the passing deployment below used the
historical Futura override. The branch is intended for a dedicated upstream
pull request after deployment verification.

## Verification and limits

With the historical Rocky 8 override, deployment passed with `ok=49`,
`changed=18`, `unreachable=0`, `failed=0`, `skipped=29`; the drop-in was
absent and all six containers ran. The Rocky 9 run passed while retaining the
drop-in. This finding is recorded as reproduced only on Rocky 8.10 based on
these tests.

The clean Rocky Linux 8.10 data-serving baseline on 2026-10-29 reproduced
the same failure at `Start THREDDS container` with the same recap:
`ok=19`, `changed=11`, `unreachable=0`, `failed=1`, `skipped=11`. With the
historical Rocky 8 override, the configured synthetic dataset was served from
both THREDDS and fileserver; see the
[data-serving test results](../ansible-deployment-test-results.md#rocky-linux-810-data-serving-scenario--2026-10-29).
