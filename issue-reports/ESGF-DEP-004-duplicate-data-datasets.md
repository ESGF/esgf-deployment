# ESGF-DEP-004 — Duplicate `data_datasets` key in upstream example

## Summary

- **Classification:** Configuration defect.
- **Observed scope:** Upstream source inspection; not specific to a guest OS.
- **Status:** Duplicate definition confirmed; the example is consolidated on
  branch `fix/ESGF-DEP-004-unique-data-datasets`, commit `d2fea8f`.
  Playbook syntax and dataset variable resolution pass. The branch is pushed;
  full deployment verification is pending.

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

The candidate branch retains one active `data_datasets` mapping and presents
the S3 configuration as a commented example, so the two mutually exclusive
examples cannot overwrite each other. Add YAML linting that rejects duplicate
keys to catch similar defects.

## Verification and limits

The original duplicate remains confirmed by source inspection. The
consolidated example resolves the two intended local datasets; a full
deployment using customized data-node variables remains untested.
