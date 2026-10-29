# ESGF-DEP-003 — Unchanged playbook rerun recreates running containers

## Summary

- **Classification:** Availability/idempotence issue; not a failed
  deployment.
- **Observed scope:** Rocky Linux 8.10 and Rocky Linux 9.8.
- **Status:** Reproduced on both; not fixed in FUTURA.

## Evidence and cause

An unchanged second Ansible run replaced all six service containers
(`proxy`, `thredds`, `fileserver`, `solr-master`, `solr-slave`, and `search`).
The container IDs changed between runs, confirming recreation. This can
briefly interrupt service. The Rocky 8 repeat completed with
`ok=47`, `changed=6`, `unreachable=0`, `failed=0`, `skipped=31`; the Rocky 9
repeat completed with `ok=50`, `changed=10`, `unreachable=0`, `failed=0`,
`skipped=30`.

The upstream `docker_container` tasks specify `recreate: yes`, which requests
container recreation rather than relying on whether configuration or image
state changed. On Rocky Linux 9, Docker setup command tasks also report
changes on repeated runs and trigger a Docker restart.

## Fix status and proposed upstream change

No fix was applied: the upstream checkout was not modified, and the
OS-specific FUTURA overrides do not change the unconditional recreation
behavior.

Remove unconditional `recreate: yes` from the affected container tasks and
allow the Docker module to replace containers when image or configuration
actually changes. Ensure pull behavior remains explicit for mutable tags,
and make Docker setup tasks idempotent so they do not trigger needless daemon
restarts.

## Verification and limits

Both repeat runs completed and the service endpoints passed checks afterward.
The duration of service interruption was not measured.

The Rocky Linux 8.10 data-serving repeat on 2026-10-29 also changed all six
container IDs (`ok=47`, `changed=6`, `unreachable=0`, `failed=0`,
`skipped=31`). The dataset catalog and fileServer continued to return HTTP
200 after the containers were recreated. The interruption duration was not
measured in this run either.
