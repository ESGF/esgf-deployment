# ANS-004 — Duplicate `data_datasets` key in upstream example

## Summary

- **Classification:** Configuration defect.
- **Observed scope:** Upstream source inspection; not specific to a guest OS.
- **Status:** Duplicate definition confirmed; runtime behavior has not been
  tested in a customized data-node deployment.

## Evidence and risk

The upstream file `esgf-docker/deploy/ansible/group_vars/data.yml` defines
`data_datasets` more than once. YAML mappings are expected to have unique
keys, but loaders can handle duplicates differently; common loaders may
silently keep only one value. Consequently, a user can lose part of the
intended dataset configuration without an obvious error.

The deployment tests did not load this file or configure data mounts, so
impact on an actual customized data node was not verified.

## Fix status and proposed upstream change

No upstream files were edited and no FUTURA fix is recorded. Combine all
dataset examples into a single `data_datasets` sequence, or remove the
duplicate unused key. Add YAML linting that rejects duplicate keys to catch
similar defects.

## Verification and limits

The duplicate was confirmed by source inspection only. Validate the corrected
file with a duplicate-key-aware YAML linter and an Ansible run that resolves
the expected dataset list before treating runtime behavior as verified.
