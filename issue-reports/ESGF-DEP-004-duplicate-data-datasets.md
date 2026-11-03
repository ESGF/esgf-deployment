# ESGF-DEP-004 — Duplicate `data_datasets` key in upstream example

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

The deployment tests do not load this upstream example file. The Rocky Linux
8.10 data-serving test on 2026-10-29 used a separate Futura inventory
`group_vars/data.yml` with one `data_datasets` key; it successfully generated
a THREDDS dataset catalog and served a synthetic NetCDF file. This validates
the single-key configuration path, not how Ansible/YAML handles the duplicate
key in the upstream example.

## Fix status and proposed upstream change

No fix branch has been created in the `esgf-docker` fork. Combine all dataset
examples into a single `data_datasets` sequence, or remove the duplicate
unused key. Add YAML linting that rejects duplicate keys to catch similar
defects.

## Verification and limits

The duplicate remains confirmed by source inspection; its runtime handling
is untested. Validate the upstream file with a duplicate-key-aware YAML
linter and an Ansible run that resolves the expected dataset list before
treating this finding as resolved.
