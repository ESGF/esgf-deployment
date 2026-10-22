# ESGF-NG site deployment requirements

## Overview

This document describes the requirements and information needed to deploy an ESGF-NG node using the Ansible deployment provided by [`esgf-docker`](https://github.com/ESGF/esgf-docker). The current FUTURA test setup and observed issues are documented in the [ESGF Ansible deployment issue log](./ansible-deployment-report.md).

The deployment runs ESGF-NG services as Docker containers on one or more site-provided hosts.

The purpose of this document is to identify what a site administrator needs to provide or configure before deployment, as well as known assumptions, limitations and gaps in the current deployment process.

---

## Deployment prerequisites

Before starting the deployment, the site administrator needs to provide or confirm the following.

### Site infrastructure

* Hostname(s) / FQDN(s)
* Target host(s) for the deployment
* DNS configuration
* Firewall configuration
* Dataset storage paths
* SSL certificate and private key, if HTTPS is enabled

### ESGF configuration

* Node type(s): data, index, or both 
* Dataset definitions to expose
* ESGF Docker image version/tag to deploy (latest by default)
* External Solr endpoint(s), if applicable

The site administrator also needs to have the required access and permissions to configure the target host(s).

---

## Infrastructure requirements

### Deployment topology

The deployment can be configured with hosts providing:

* Data services
* Index services
* Both data and index services

### Compute

No minimum CPU requirements are currently documented.

### Memory

No minimum memory requirements are currently documented.

### Host operating system

The operating-system compatibility matrix is not yet formally defined. The
current FUTURA deployment test has been run on Rocky Linux 8.10 only; this
single test does not establish support for other versions.

The following distributions are candidates for testing, based on the
`esgf-docker` deployment role's platform conditions and project plans:

* Rocky Linux 8
* AlmaLinux 8
* AlmaLinux 9
* RHEL 8
* RHEL 9

The list is not a guarantee of compatibility. Record each tested distribution
and version in the issue log and treat untested platforms as unverified.

The host operating system should be installed and configured before running the ESGF deployment.

### Network and DNS

The following inbound ports need to be accessible for the Docker host:

* `80` — HTTP
* `443` — HTTPS

The public ESGF service hostname(s) must resolve to the appropriate host(s).

The deployment may also require outbound network connectivity to retrieve Docker images and communicate with external services.

Network access must also be available between the Ansible deployment machine and the target host(s).

### TLS / SSL certificates

If HTTPS is enabled, the site administrator must provide an SSL certificate and private key.

### Storage

The upstream deployment documentation does not currently establish a
minimum-free-disk requirement. Confirm the required disk capacity against
container image sizes, writable container data, logs, and the intended dataset
storage before deployment. The FUTURA Rocky Linux 8.10 test VM had a 30 GB
root disk; that test does not establish a minimum.

Dataset storage must be available on the Docker host and mounted into the relevant containers.

---

## Software and access requirements

### Ansible

Ansible must be installed on the machine from which the deployment is executed.

The Ansible deployment is provided [here](https://github.com/ESGF/esgf-docker/blob/master/docs/deploy-ansible.md) in the `esgf-docker` repository.

### Docker

The ESGF-NG services are deployed as Docker containers on the target host(s).

Docker is installed and configured by the Ansible deployment.

### Access and permissions

The deployment requires root or sudo access to the target Docker host(s).

After deployment, the required ESGF service ports must be accessible according to the site's firewall and network policies.

---

## Ansible configuration


The deployment requires site-specific Ansible configuration, including an inventory defining the target hosts.

The inventory needs to identify the hosts providing the relevant services, such as data and index services.

Site-specific variables can be configured using `group_vars` and `host_vars`.

The deployment is run using the Ansible playbook provided in the repository.

---

## Validation

After deployment, the administrator should verify that the deployment is operational.

Basic validation should include:

* Confirm that the expected Docker containers are running.

* Confirm that the THREDDS endpoint is accessible:

  `http(s)://<data-host>/thredds`

* Confirm that configured datasets are visible.

* Confirm that a test dataset can be accessed or downloaded.

* Confirm that index/search functionality is working when an index service is deployed.

---

## Known issues and deployment caveats

The following issues have been observed or identified during the FUTURA
deployment work. See the [issue log](./ansible-deployment-report.md)
for evidence, workarounds, verification, and scope:

* On Rocky Linux 8.10, the upstream Docker role's `MountFlags=shared` systemd
  drop-in prevented containers from starting. The FUTURA test removes this
  drop-in; behavior on other operating systems has not been tested.
* The upstream proxy role's TLS file-permission task fails when no certificate
  and private key are configured. The FUTURA test guards this task when TLS
  files are absent. HTTPS and certificate provisioning remain untested.
* Re-running the upstream playbook recreates service containers, causing
  service interruption even when configuration is unchanged.
* The upstream example `group_vars/data.yml` defines `data_datasets` twice.
  Loading it can replace the earlier dataset list with the later one.
* An initial check-mode run against a fresh host failed because the auth role
  queried Docker before Docker had been installed and started.

The Vagrant example's CentOS 7 repository availability should be checked
separately if that example is used; it was not part of the Rocky Linux 8.10
deployment test.

---

## Open questions, gaps and things to explore

The following areas are currently unclear, site-specific, undocumented or should be validated before the deployment can be considered fully reusable by another site.

### Platform compatibility

* What Linux distributions and versions actually work?
* Which distributions and versions have been tested?
* What should the supported/tested compatibility matrix be?

### Resource requirements

* What are the minimum/recommended CPU requirements?
* What are the minimum/recommended memory requirements?

### Deployment topology

* What topology should be recommended for a new site?
* When should data and index services be deployed on separate hosts?

### Ansible and Docker

* Which Ansible versions are supported?
* Which Docker versions are supported?

### Container images and releases

* Which ESGF Docker image version should a production site deploy?

### Operations

* How should an existing deployment be upgraded?
* How should Docker images and services be updated?
