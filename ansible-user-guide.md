# Quick guide: deploy an ESGF data node with Ansible

This guide covers a basic single-host deployment using
[`esgf-docker`](https://github.com/ESGF/esgf-docker). For advanced options,
see the [official Ansible guide](https://github.com/ESGF/esgf-docker/blob/master/docs/deploy-ansible.md).

## What you need

- A Linux host compatible with the deployment's `dnf`/systemd tasks, reachable
  by SSH, with a fully qualified hostname and root/sudo access. The containers
  need about 4 GB free disk space; THREDDS needs about 4 GB RAM minimum, with
  16 GB recommended. Provide additional storage for your data.
- Inbound HTTP/HTTPS access (ports 80/443), DNS for the public hostname, and
  outbound access to download container images.
- A separate control machine with Ansible installed and SSH access to the
  host. Ansible installs and configures Docker on the host.
- A certificate and private key if you plan to enable HTTPS. The default proxy
  configuration is HTTP-only.

The supported OS matrix is not formally defined. FUTURA has tested Rocky
Linux 8.10 and 9.8; this does not guarantee compatibility. See the
[test results and known issues](./ansible-deployment-test-results.md).
Some known fixes are available on FUTURA fork branches but still need
deployment verification; check the results before choosing a code revision.

## What gets deployed

Ansible installs Docker, creates a shared Docker network, then starts the
containers selected by your inventory:

| Container | Deployed when | Purpose |
|---|---|---|
| `proxy` (Nginx) | Every host | Public web entry point; routes requests to the services below. HTTP is the default; HTTPS needs TLS configuration. |
| `thredds` | Host is in `[data]` | Builds dataset catalogs and provides THREDDS/OPeNDAP access. |
| `fileserver` (Nginx) | Host is in `[data]` | Serves dataset files directly over HTTP. |
| `opa` and `auth` | Authentication is enabled | Open Policy Agent and the ESGF authentication service; disabled by default. |
| `logstash` | Log forwarding is enabled on a data host | Forwards service access logs to the configured statistics endpoint; disabled by default. |

This guide covers a data node only. Add hosts to the `[data]` inventory group;
configure the services you need for data serving only.

## Configure and deploy

On the control machine, clone `esgf-docker` and create a configuration
directory outside that checkout:

```sh
git clone https://github.com/ESGF/esgf-docker.git
mkdir -p ~/esgf-config/group_vars
```

Create `~/esgf-config/inventory.ini` (replace the hostname and user):

```ini
[data]
esgf.example.org ansible_user=deploy
```

For a data node, create `~/esgf-config/group_vars/data.yml`. The host data
directory must already exist and be readable by the containers:

```yaml
data_mounts:
  - host_path: /srv/esgf-data
    mount_path: /data

data_datasets:
  - name: Example dataset
    path: example
    location: /data/example
```

Check connectivity, then validate and run the playbook from the `esgf-docker`
directory:

```sh
ansible all -i ~/esgf-config/inventory.ini -m ping
ansible-playbook -i ~/esgf-config/inventory.ini deploy/ansible/playbook.yml --syntax-check
ansible-playbook -i ~/esgf-config/inventory.ini deploy/ansible/playbook.yml -K
```

Ansible connects over SSH, installs/configures Docker, prepares service
configuration and storage mounts, then starts the containers selected by the
inventory groups. It does **not** provision the machine, mount or populate
your data storage, configure public DNS, or open the site's firewall.

After deployment, check that the containers are running and visit
`http://esgf.example.org/thredds` (or HTTPS if configured). For production,
select a pinned immutable image tag rather than the default mutable `latest`.
