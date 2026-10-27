# ANS-001 — Docker mount setting prevents container startup

## Summary

- **Classification:** Runtime/deployment issue.
- **Observed scope:** Reproduced on a clean Rocky Linux 8.10 host; not
  reproduced on Rocky Linux 9.8.
- **Status:** A Rocky 8-specific workaround is in FUTURA. No Rocky 9
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

## Fixes and proposed upstream change

The Rocky 8 override removes the drop-in, reloads systemd when it is removed,
and restarts Docker only in that case. It also avoids repeated repository and
package changes that would otherwise restart Docker unnecessarily.

The upstream role should validate whether the mount override is required and
avoid applying it on host/runtime combinations where it prevents containers
from starting. A fix should be tested across supported host OS/runtime
combinations before generalizing the workaround.

FUTURA override: [`rocky8 Docker role`](../ansible/overrides/rocky8/roles/docker/tasks/main.yml).

## Verification and limits

After applying the Rocky 8 override, deployment passed with `ok=49`,
`changed=18`, `unreachable=0`, `failed=0`, `skipped=29`; the drop-in was
absent and all six containers ran. The Rocky 9 run passed while retaining the
drop-in. This finding is recorded as reproduced only on Rocky 8.10 based on
these tests.
