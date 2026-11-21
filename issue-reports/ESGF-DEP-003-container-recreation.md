# ESGF-DEP-003 — Unchanged playbook rerun recreates running containers

## Summary

- **Classification:** Availability/idempotence issue; not a failed
  deployment.
- **Observed scope:** Rocky Linux 8.10 and Rocky Linux 9.8.
- **Status:** Candidate fix pushed on branch
  `fix/ESGF-DEP-003-idempotent-container-runs`, commit `69dc037`; syntax
  check passes. Deployment testing on a data-only inventory is pending.

## Evidence and cause

An unchanged second Ansible run replaced all six containers configured in
the historical test inventory, which included services outside the current
data-node scope. The container IDs changed between runs, confirming
recreation. This can briefly interrupt service. The Rocky 8 repeat completed
with `ok=47`, `changed=6`, `unreachable=0`, `failed=0`, `skipped=31`; the
Rocky 9 repeat completed with `ok=50`, `changed=10`, `unreachable=0`,
`failed=0`, `skipped=30`.

The upstream `docker_container` tasks specify `recreate: yes`, which requests
container recreation rather than relying on whether configuration or image
state changed. On Rocky Linux 9, Docker setup command tasks also report
changes on repeated runs and trigger a Docker restart.

## Fix status and proposed upstream change

The historical OS-specific FUTURA overrides do not change the unconditional
recreation behavior. The candidate branch removes forced recreation from
service-container tasks and makes Docker repository, package, and service
setup idempotent so an unchanged rerun does not trigger a Docker restart.

Remove unconditional `recreate: yes` from the affected container tasks and
allow the Docker module to replace containers when image or configuration
actually changes. Ensure pull behavior remains explicit for mutable tags,
and make Docker setup tasks idempotent so they do not trigger needless daemon
restarts.

## Verification and limits

Both repeat runs completed and the service endpoints passed checks afterward.
The duration of service interruption was not measured.

The Rocky Linux 8.10 data-serving repeat on 2026-10-29 also changed all
container IDs in that historical inventory (`ok=47`, `changed=6`,
`unreachable=0`, `failed=0`, `skipped=31`). The dataset catalog and fileServer
continued to return HTTP 200 after the containers were recreated. The
interruption duration was not measured in this run either.

The Rocky Linux 9.8 data-serving repeat on 2026-10-30 also changed all
container IDs in that historical inventory (`ok=50`, `changed=10`,
`unreachable=0`, `failed=0`, `skipped=30`). The dataset catalog, fileServer,
OPeNDAP, and health endpoints returned HTTP 200 after recreation. Interruption
duration was not measured. The candidate branch has not been run on either
OS; container identity and service continuity must be verified on an
unchanged second run before considering the fix effective.
