# ESGF Deployment

**Community resources for deploying and integrating ESGF services**

> **Status:** Preliminary draft, under review.

## Overview

This repository aims to provide a common entry point for deployment and operational resources across the ESGF community.

The goal is to make existing ESGF software and operational experience easier to reuse, helping sites deploy, configure, integrate and validate ESGF services in different environments.

Rather than replacing existing software repositories or documentation, this repository brings together deployment recipes, tested configurations, validation resources and links to authoritative upstream sources.

## Scope

The initial focus is on **ESGF Data Node capabilities**, particularly:

- **Data publication:** preparing and registering datasets in the ESGF federation.
- **Data replication:** acquiring and managing copies of datasets.
- **Data access:** hosting and serving datasets, including replicas.

Different sites may have different roles and provide different combinations of these capabilities. We do not assume a single Data Node architecture or a fixed set of services that every site must deploy.

Deployment resources should also consider dependencies on other ESGF infrastructure services, such as authentication, vocabularies, metadata catalogues and federation services.

The scope may evolve to cover additional ESGF services and deployment scenarios, depending on community needs and contributions.

## What this repository provides

The intention is to progressively collect and develop:

- References to existing ESGF software and deployment resources.
- Reusable deployment recipes and configuration examples.
- Infrastructure and integration requirements.
- CI/CD templates and deployment validation procedures.
- Container images and associated build resources, where appropriate.
- Tested versions, compatibility information and known limitations.
- Operational knowledge contributed by ESGF sites.

The repository is not limited to documentation. Deployment recipes, configurations, workflows and other reusable artefacts may also be maintained and versioned here.

Existing component-specific repositories remain the authoritative sources for their software and documentation.

This repository is not intended to replace software development, provide a central ESGF helpdesk or assume responsibility for operating services at individual sites.

## Community and contributions

This is an **ESGF community resource**, open to contributions from software developers, infrastructure operators and site administrators.

It aims to bring together deployment resources and operational experience from different ESGF sites and teams, covering different deployment technologies and approaches.

Contributions and maintenance arrangements will evolve with the participation of the wider ESGF community.

## Initial deployment resources

The first resources available in this repository document deployment and validation work using the existing [esgf-docker](https://github.com/ESGF/esgf-docker) Ansible deployment approach.

- [Ansible user guide](./ansible-user-guide.md)
- [Deployment requirements](./deploy_requirements.md)
- [Deployment test methodology](./ansible-deployment-test-methodology.md)
- [Deployment test results](./ansible-deployment-test-results.md)
- [Deployment test roadmap](./ansible-deployment-test-roadmap.md)
- [Deployment issue reports](./issue-reports/)

These documents describe deployment requirements, reproducible testing on clean virtual machines, observed results and known issues, including tests with Rocky Linux 8.10 and 9.8.

The test-specific Ansible playbooks, inventories and configurations are not currently included in this repository. The published documentation does not yet constitute a complete, self-contained deployment recipe.

Software fixes and component-specific improvements should be contributed to their upstream repositories wherever possible.

## Contributing

Contributions from the ESGF community are welcome, including deployment recipes, configuration examples, validation results and operational experience.

Contribution guidelines and maintenance arrangements will be developed with the ESGF community.

## Acknowledgements and funding

The initial development of this community repository has been supported by **FUTURA**, a project funded by the European Union's Horizon Europe research and innovation programme under Grant Agreement No. **101295156**.

The repository also builds on software, documentation and operational experience contributed by the wider ESGF community.

Funded by the European Union. Views and opinions expressed are however those of the author(s) only and do not necessarily reflect those of the European Union or the European Climate, Infrastructure and Environment Executive Agency (CINEA). Neither the European Union nor the granting authority can be held responsible for them.
