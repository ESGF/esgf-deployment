# ESGF-DEP-005 — Futura playbook does not discover upstream sibling group variables

## Summary

- **Classification:** Test/configuration caveat, not an upstream deployment
  failure.
- **Observed scope:** Futura-local playbook setup; verified on Rocky Linux
  8.10 and 9.8.
- **Status:** Inventory-local workaround verified for both data-serving
  tests. Upstream sibling group variables remain undiscovered by the
  Futura-local playbook.

## Details

The Futura playbook is stored outside the upstream `deploy/ansible`
directory. Ansible variable discovery follows the playbook/inventory
structure; it does not automatically load the upstream playbook directory's
sibling `group_vars` merely because roles are sourced from that repository.

The data-serving scenario keeps its dataset and mount configuration in
Futura's inventory-local `group_vars/data.yml`, so the test playbook resolves
the intended values without relying on upstream sibling files. Both Rocky
Linux 8.10 and 9.8 deployments verified this configuration path.

## Resolution for customized deployments

Review the needed upstream variables and either pass reviewed variable files
explicitly or place them within a directory structure Ansible discovers for
the Futura playbook/inventory. Before deployment, inspect the resolved
variables and verify that expected host/group values are present; do not
assume the upstream files were loaded.

## Verification and limits

On 2026-10-29, Futura placed a single-key data configuration in
`ansible/group_vars/data.yml`, beside the inventory, and
`ansible-inventory --host node01` showed the expected `data_mounts` and
`data_datasets`. The prepared fixture was then visible in both THREDDS and
fileserver containers and appeared in the generated dataset catalog.
On 2026-10-30, the same resolved variables and data-serving behavior were
verified on Rocky Linux 9.8. Therefore inventory-local group variables work
for this Futura setup; this
does not change Ansible's discovery behavior for the upstream sibling
`group_vars` directory.

The repeatable baseline-first process is described in the
[`Ansible deployment test methodology`](../ansible-deployment-test-methodology.md).
