# ESGF-DEP-005 — Futura playbook does not discover upstream sibling group variables

## Summary

- **Classification:** Test/configuration caveat, not an upstream deployment
  failure.
- **Observed scope:** Futura-local playbook setup for both OS baseline runs.
- **Status:** Documented; no external group variables were loaded.

## Details

The Futura playbook is stored outside the upstream `deploy/ansible`
directory. Ansible variable discovery follows the playbook/inventory
structure; it does not automatically load the upstream playbook directory's
sibling `group_vars` merely because roles are sourced from that repository.

The baseline deployments intentionally used role defaults and no external
datasets. This kept the initial runs focused on default deployment behavior,
but means upstream group-variable configuration, data mounts, and dataset
settings were not exercised.

## Resolution for customized deployments

Review the needed upstream variables and either pass reviewed variable files
explicitly or place them within a directory structure Ansible discovers for
the Futura playbook/inventory. Before deployment, inspect the resolved
variables and verify that expected host/group values are present; do not
assume the upstream files were loaded.

## Verification and limits

Both baseline plays ran without upstream `group_vars`. No variables were
staged or altered, and data mount/dataset behavior remains unvalidated.

The repeatable baseline-first process is described in the
[`Ansible deployment test methodology`](../ansible-deployment-test-methodology.md).
