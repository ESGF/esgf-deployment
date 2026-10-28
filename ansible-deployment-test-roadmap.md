# Ansible Deployment Test Roadmap

FUTURA tests the upstream `esgf-docker` Ansible deployment across useful
configurations and environments, then reports reproducible evidence and
proposed upstream fixes. Existing Rocky Linux 8.10/9.8 tests cover HTTP-only
deployments with no datasets and mutable `latest` image tags.

## Priorities

| Priority | Scenario | Key validation |
|---:|---|---|
| 1 | **Data-serving node** | Configure one host data mount and representative dataset. Verify mount visibility, THREDDS catalog, file access, and search behavior. Validate `data_datasets` first ([ESGF-DEP-004](./issue-reports/ESGF-DEP-004-duplicate-data-datasets.md)). |
| 2 | **TLS-enabled deployment** | Use a valid certificate/key; verify HTTPS, certificate presentation, and service endpoints. Keep separate from HTTP-only testing ([ESGF-DEP-002](./issue-reports/ESGF-DEP-002-proxy-tls-permissions.md)). |
| 3 | **Repeat and image-update behavior** | Compare image/container IDs and endpoint availability across an unchanged rerun and a deliberate image update. Record Docker restarts, recreation, interruption, and recovery ([ESGF-DEP-003](./issue-reports/ESGF-DEP-003-container-recreation.md)). |
| 4 | **Additional host OSes** | Add distributions supported by `esgf-docker` or needed for a real deployment. Start clean, run upstream-only baselines, and isolate OS-specific fixes. |

## Test and reporting rules

- Change one major configuration dimension per scenario. Start with a clean
  guest and unmodified upstream roles; capture baseline failures before fixes.
- Keep fixes isolated by OS/scenario and do not modify the upstream checkout.
- Record full Ansible recaps, services/containers, endpoints, repeat-run
  behavior, and untested areas.
- After each test, update the general deployment report, affected detailed
  finding reports, and OpenTofu tested-environments report as described in
  the [test methodology](./ansible-deployment-test-methodology.md).
- Test each scenario on one representative environment first; repeat on
  another OS only when the comparison is useful. Do not build a full
  OS-by-configuration matrix.
