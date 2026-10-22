# ESGF Deployment

**Community resources for deploying and integrating ESGF services**

> **Status:** Initial proposal, under discussion.

## Overview

This repository aims to provide a common entry point for deployment and operational resources across the ESGF community.

The goal is to make existing ESGF software and operational experience easier to reuse, helping sites deploy, configure, integrate and validate ESGF services in different environments.

Rather than replacing existing software repositories or documentation, this repository will bring together deployment recipes, tested configurations, validation resources and links to authoritative upstream sources.

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

Existing component-specific repositories remain the authoritative sources for their software and documentation.

This repository is not intended to replace software development, provide a central ESGF helpdesk or assume responsibility for operating services at individual sites.

## Community and contributions

This is intended to be an **ESGF community resource**, open to contributions from software developers, infrastructure operators and site administrators.

The initial work is being coordinated through **FUTURA Task 8.3 (Setup DevOps)**, led by CSIC with the participation of the T8.3 partners.

FUTURA T8.3 will initially focus on reusable deployment resources for publication, replication and data access, building on existing deployments and operational experience.

The broader community repository may evolve beyond these initial activities, with contributions and maintenance responsibilities agreed among the participating teams.

## Getting started

The `esgf-docker` Ansible deployment results are documented in the
[deployment issue log](./ansible-deployment-report.md). The repeatable
clean-VM, upstream-baseline-first process is in the
[Ansible test methodology](./ansible-test-methodology.md). The Futura-local
playbook, inventory, and OS-specific role overrides are in [`ansible/`](./ansible/).

The upstream `esgf-docker` checkout is treated as a read-only reference. See
the reports for reproducible commands and Rocky Linux 8.10/9.8 test results.

## Contributing

*Contribution guidelines and repository organisation will be agreed with

## Acknowledgements and funding

The initial development and coordination of this community repository are supported by **FUTURA (Task 8.3, Setup DevOps)**, a project funded by the European Union's Horizon Europe research and innovation programme under Grant Agreement No. **101295156**.

This repository is intended as a community resource for ESGF, with contributions from different projects, institutions and ESGF teams.

Funded by the European Union. Views and opinions expressed are however those of the author(s) only and do not necessarily reflect those of the European Union or the European Climate, Infrastructure and Environment Executive Agency (CINEA). Neither the European Union nor the granting authority can be held responsible for them.