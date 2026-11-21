# Ansible Deployment Test Roadmap

FUTURA tests the upstream `esgf-docker` Ansible deployment across useful
configurations and environments, records reproducible evidence, and develops
isolated candidate fixes in the FUTURA fork for contribution upstream. The
historical data-serving checks exist for Rocky Linux 8.10 and 9.8; repeat
testing with a data-only inventory is still needed.

## Priorities

| Priority | Scenario | Key validation |
|---:|---|---|
| 1 | **Data-serving node** | Rocky Linux 8.10 and 9.8 have historical data-serving results. Configure one host data mount and representative dataset. Verify mount visibility, THREDDS catalog, file access, and OPeNDAP. Validate `data_datasets` first ([ESGF-DEP-004](./issue-reports/ESGF-DEP-004-duplicate-data-datasets.md)). |
| 2 | **TLS-enabled deployment** | Use a valid certificate/key; verify HTTPS, certificate presentation, and service endpoints. Keep separate from HTTP-only testing ([ESGF-DEP-002](./issue-reports/ESGF-DEP-002-proxy-tls-permissions.md)). |
| 3 | **Repeat and image-update behavior** | Compare image/container IDs and endpoint availability across an unchanged rerun and a deliberate image update. Record Docker restarts, recreation, interruption, and recovery ([ESGF-DEP-003](./issue-reports/ESGF-DEP-003-container-recreation.md)). |
| 4 | **Additional host OSes** | Add distributions supported by `esgf-docker` or needed for a real deployment. Start clean, run upstream-only baselines, and create a dedicated fork branch for each actionable finding. |

## Test and reporting rules

- Change one major configuration dimension per scenario. Start with a clean
  guest and unmodified upstream roles; capture baseline failures before fixes.
- For each actionable `esgf-docker` issue, create a dedicated fix branch in
  the FUTURA fork. Keep the upstream repository read-only. Leave changes
  uncommitted and unpushed for user review; commit or push only when explicitly
  requested. Use `origin` for the fork and `upstream` for the canonical GitHub
  repository.
- Record full Ansible recaps, services/containers, endpoints, repeat-run
  behavior, and untested areas.
- After each test, update the general deployment report, affected detailed
  finding reports as described in the
  [test methodology](./ansible-deployment-test-methodology.md). Keep
  provisioning-tool failures out of ESGF deployment reports; mark scenarios
  without deployment evidence as not tested.
- Distinguish historical local override results from verification performed
  against a fix branch. Record the branch and working-tree status in the
  issue report; record a commit only after the user requests it.
- Test each scenario on one representative environment first; repeat on
  another OS only when the comparison is useful. Do not build a full
  OS-by-configuration matrix.
