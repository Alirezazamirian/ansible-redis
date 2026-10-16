# Ansible Redis Cluster

Inventory-driven automation for deploying Redis Cluster with sharding, replication, TLS, ACL-based authentication, and a Prometheus-compatible Redis Exporter. Includes package repository configuration and Ansible Vault integration for environment-specific credentials.

The deployment uses native Redis Cluster rather than Redis Sentinel for sharding and failover. The supplied production inventory contains six Redis nodes and requests one replica per primary.

> Deployment status: the attached implementation includes integration gaps and security-sensitive defaults. Complete the [pre-deployment checklist](#pre-deployment-checklist) before running against production. Configuration switches describe intended capabilities, not a guarantee that every combination currently works.

## Contents

- [Features](#features)
- [Architecture and topology](#architecture-and-topology)
- [Project layout](#project-layout)
- [Requirements](#requirements)
- [Quick start](#quick-start)
- [Configuration](#configuration)
- [TLS and certificates](#tls-and-certificates)
- [Authentication and secrets](#authentication-and-secrets)
- [Monitoring](#monitoring)
- [Validation and operations](#validation-and-operations)
- [Pre-deployment checklist](#pre-deployment-checklist)
- [License and author](#license-and-author)

## Features

- Redis Cluster bootstrapping from the `redis_cluster` inventory group.
- Configurable replicas per primary through `redis_cluster_each_master_replicas`.
- Native cluster sharding and replica promotion without a Sentinel dependency, subject to correct configuration and Redis quorum requirements.
- Client-to-Redis mutual TLS, with a generated root CA and signed node, client, and controller certificates.
- TLS configuration for node-to-node cluster traffic and replication.
- ACL users with configurable command permissions, key patterns, and enable/disable flags.
- Password-hashing switches for Redis ACLs and exporter HTTP Basic authentication; see the implementation caveat below for Redis ACL hashing.
- Redis Exporter with Redis authentication, TLS client credentials, an HTTPS metrics endpoint, and optional HTTP Basic authentication.
- Configurable package mirrors and exporter artifact downloads, including HTTPS and repository credentials.
- Ansible Vault-backed credentials shared across inventory, repository, Redis, and exporter configuration.
- systemd service templates and SELinux-aware file labels.
- Redis persistence, memory, replication, latency, and cluster tuning through inventory variables.

## Architecture and topology

```text
Ansible controller
  |-- Configure package repositories
  |-- Install Redis and render configuration / ACLs
  |-- Generate and distribute certificates
  |-- Bootstrap Redis Cluster using redis-cli
  `-- Install and configure Redis Exporter

Application clients -- TLS + ACL authentication --> Redis Cluster
                                                      |
                                      Sharded primaries + replicas
                                      Cluster bus / replication TLS
                                                      |
Prometheus -- HTTPS + HTTP Basic auth --> Redis Exporter
                                            |
                                      TLS + Redis ACL user
                                            |
                                           Redis
```

The roles discover node endpoints from inventory rather than hard-coding a six-node cluster creation command. However, inventory-driven does not mean every host count provides a valid or highly available cluster.

Redis documents a minimum of three primaries for a working cluster and recommends six nodes—three primaries and three replicas—for a basic replicated deployment. Increasing the number of nodes requires a compatible primary/replica topology and appropriate failure-domain placement. A single-node deployment is not highly available.

The supplied `is_master` host variables do not pin the roles assigned by `redis-cli --cluster create`: bootstrap passes all inventory endpoints to Redis with a replica count. Verify actual assignments after creation.

Sentinel-related templates and variables exist, but `redis_sentinel_enable` is `false` in the supplied configuration. Treat Sentinel as a separate, unvalidated path rather than part of the standard deployment.

## Project layout

```text
.
├── ansible.cfg
├── galaxy.yml
├── inventories/
│   ├── production/
│   │   ├── hosts.yml
│   │   ├── group_vars/
│   │   │   ├── all/
│   │   │   │   ├── all.yml
│   │   │   │   └── vault.yml.example
│   │   │   ├── redis_cluster.yml
│   │   │   └── observability.yml
│   │   └── host_vars/
│   └── staging/
├── playbooks/
│   └── production/
│       ├── all.yml
│       └── cache.yml
└── roles/
    ├── redis/
    ├── redis-exporter/
    └── repos/
```

| Component | Responsibility |
| --- | --- |
| `repos` | Render local package repositories and refresh package metadata. |
| `redis` | Install Redis, render configuration and ACLs, generate TLS material, manage services, and bootstrap the cluster. |
| `redis-exporter` | Download the exporter artifact, create its service account, render environment/systemd files, and configure certificates. |
| `cache.yml` | Gather controller and managed-host facts before the deployment plays. |
| `all.yml` | Execute fact gathering, repository setup, Redis deployment, and exporter deployment in order. |

Most project configuration lives in inventory group variables; the role defaults are largely empty. Staging is a scaffold and requires its own variables, credentials, and playbook setup.

## Requirements

### Controller

- Python 3, a virtual environment, and an Ansible release compatible with the modules used by this project. A tested minimum version is not established by the supplied metadata.
- `community.crypto` and `ansible.posix` collections.
- Python `cryptography`, plus `passlib` and a bcrypt backend for the configured password filters.
- A TLS-capable `redis-cli` reachable through `PATH`; cluster creation runs on the controller.
- SSH access and privilege escalation on managed hosts, plus local privilege escalation where delegated tasks require it.
- Writable `roles/redis/files/` and `roles/redis-exporter/files/` directories for generated certificate material.
- A `firewall` role available on Ansible's role search path. Both Redis roles include it, but it is not included in the supplied tree.

The certificate tasks explicitly reference a repository-local `venv/bin/pip`; use that virtual environment location unless you change the tasks. They also request the controller package `openssh-client`; adapt that package name to the controller OS if necessary and ensure OpenSSL tooling is available.

### Managed hosts

- Linux with systemd and Ansible-compatible Python.
- Access to the configured package mirror and exporter artifact server.
- Redis packages compatible with the directives in `redis.conf.j2`. Redis itself is not version-pinned by the supplied install tasks.
- Appropriate network routes, name resolution, firewall policies, and SELinux configuration.

Redis package installation has Debian and Red Hat branches, but the supplied mirror paths target Rocky Linux and certificate/configuration tasks reference `ansible_ens33`. Validate distribution and interface assumptions before using another environment.

## Quick start

The following workflow assumes the pre-deployment checklist has been addressed. Run commands from the repository root.

### 1. Prepare the controller

```bash
python3 -m venv venv
source venv/bin/activate
python -m pip install --upgrade pip
python -m pip install ansible cryptography passlib bcrypt
ansible-galaxy collection install community.crypto ansible.posix
mkdir -p logs roles/redis/files roles/redis-exporter/files
```

Provide the missing `firewall` role separately. Installing the collections does not supply it.

Review `ansible.cfg`: the supplied file contains environment-specific inventory, SSH key, user, and port settings. All commands below explicitly select the production inventory rather than relying on its default inventory path.

### 2. Configure inventory

Edit:

- `inventories/production/hosts.yml`: Redis members and optional client hosts.
- `inventories/production/host_vars/*.yml`: host addresses, SSH settings, privilege escalation, and host-specific values.
- `inventories/production/group_vars/all/all.yml`: shared switches and repository settings.
- `inventories/production/group_vars/redis_cluster.yml`: Redis, TLS, ACL, exporter, and repository configuration.

The example inventory uses SSH port `5119`; change it to match your hosts. Provide connection details for client hosts too if you keep them in the inventory.

### 3. Create the encrypted secrets file

Use the complete schema in `vault.yml.example` as your starting point:

```bash
cp inventories/production/group_vars/all/vault.yml.example \
   inventories/production/group_vars/all/vault.yml
chmod 600 inventories/production/group_vars/all/vault.yml
# Edit vault.yml locally and replace the placeholders before encrypting.
ansible-vault encrypt inventories/production/group_vars/all/vault.yml
```

Subsequent edits:

```bash
ansible-vault edit inventories/production/group_vars/all/vault.yml
```

Never commit the populated plaintext file or a Vault password file.

### 4. Check inventory and syntax

```bash
ansible-inventory -i inventories/production/hosts.yml --graph --ask-vault-pass

ansible-playbook -i inventories/production/hosts.yml \
  playbooks/production/all.yml --syntax-check --ask-vault-pass
```

A successful syntax check is not proof that templates, certificates, authentication, or cluster bootstrap will work.

### 5. Deploy

```bash
ansible-playbook -i inventories/production/hosts.yml \
  playbooks/production/all.yml --ask-vault-pass
```

The production playbook currently applies the exporter to `all`, not just `redis_cluster`. Its service requires `redis.service`, and exporter variables are defined for the Redis group. Restrict that play to suitable Redis hosts or provide a deliberate configuration for every other targeted host before running it.

## Configuration

Values below describe the supplied production inventory, not universal role defaults.

### Main switches

| Variable | Supplied value | Purpose |
| --- | --- | --- |
| `redis_tls_enable` | `true` | Enable Redis client TLS and require client certificates in the template. |
| `redis_cluster_tls_enable` | `true` | Feed the cluster/replication TLS configuration. |
| `redis_security_acl_enable` | `true` | Render the Redis ACL file. |
| `redis_master_auth_enable` | `true` | Configure replication authentication. |
| `redis_security_use_hash_password` | `true` | Select the current ACL hashing branch; requires correction before use. |
| `redis_cluster_each_master_replicas` | `1` | Replica count passed to cluster creation. |
| `redis_sentinel_enable` | `false` | Keep Sentinel outside the standard deployment. |
| `redis_exporter_use_tls` | `true` | Enable HTTPS for exporter metrics. |
| `redis_exporter_use_basic_http_auth` | `true` | Enable exporter HTTP Basic authentication. |
| `redis_exporter_use_hash_pass` | `true` | Intended selection of hashed exporter HTTP credentials; review both emitted password variables when disabling it. |
| `redis_exporter_use_environment_file` | `true` | Declared setting; current tasks choose the environment file by template existence rather than this flag. |

### Paths and ports

| Setting | Supplied value |
| --- | --- |
| Redis configuration | `/etc/redis/redis.conf` |
| Redis ACL file | `/etc/redis/users.acl` |
| Redis data | `/var/lib/redis` |
| Redis certificates | `/etc/redis/certs` |
| Redis logs | `/var/log/redis` |
| Redis non-TLS port | `6379`, disabled in `redis.conf` when client TLS is enabled |
| Redis TLS client port | `6380` |
| Cluster bus settings | `16379` and `16380`; reconcile listening and announced ports before deployment |
| Exporter metrics port | `9121` |
| Exporter environment file | `/etc/default/redis_exporter` |
| Exporter certificates | `/etc/redis-exporter/certs` |
| Exporter binary | `/usr/local/bin/redis_exporter` |

The template explicitly sets `cluster-port` from `cluster_listen_port`, while announce settings use the cluster TLS port in TLS mode. Do not assume that enabling TLS automatically makes the configured bus and advertised client endpoints consistent.

### Package repositories

`repos_info.mirror` controls package mirror address, HTTPS, authentication, and optional controller repository configuration. `repos_info.ftp` controls the HTTP(S) artifact endpoint used for exporter downloads; despite the variable name, its constructed URL is HTTP or HTTPS, not the FTP protocol.

`local_repo_base_addrs` defines mirror paths. Individual dictionaries such as `repos_redis`, `repos_baseos`, and `repos_appstream` control enablement and priority.

> The repository role deletes existing files from `/etc/yum.repos.d/` or `/etc/apt/sources.list.d/` before writing its own templates. Back up existing configuration and review the full template set first. The current implementation disables package signature checks and HTTPS certificate verification in relevant paths.

Mirror URL templates only replace `@` with `%40` in passwords; they do not perform general URL encoding. Review credential handling before using special characters or embedding credentials in generated URLs.

### Redis tuning

`redis_conf_options` controls persistence, memory limits, eviction, replication, I/O threads, logging, and latency settings. `redis_cluster_conf_options` controls cluster behavior and advertised endpoints.

The supplied inventory enables AOF with `everysec` fsync, configures RDB snapshots, and derives the memory limit from 80% of reported RAM. Review these settings against application durability, memory overhead, and recovery requirements.

When overriding dictionaries, preserve required entries: the templates access many keys directly, and the configuration does not enable dictionary merging.

## TLS and certificates

The Redis role generates a root CA on the controller, signs Redis node certificates, generates controller credentials for bootstrap, and creates certificates for inventory members in `clients` when that group exists. It copies Redis node certificates and the CA to Redis hosts; application-side installation of client credentials remains a separate integration step.

The Redis template configures:

- `tls-auth-clients yes` when client TLS is enabled.
- TLS 1.2 and TLS 1.3.
- Server and outbound-client certificate/key paths.
- `tls-cluster` and `tls-replication` from the cluster TLS configuration.
- `port 0` to disable plaintext client connections while TLS is enabled.

The exporter has two distinct TLS relationships:

1. Exporter to Redis: a client certificate signed by the Redis CA.
2. Prometheus to exporter: a separate self-signed web certificate in the current implementation.

Trust the correct certificate for each connection. The generated exporter web certificate SANs currently use controller-oriented identities rather than each exporter's scrape address; correct them before enabling strict hostname verification.

Certificate subjects, DNS names, interface facts, and passphrase references contain environment-specific assumptions. Review them before use.

> Certificate tasks remove existing files from role certificate directories and selected managed certificate locations. Rerunning them is not a safe, transparent certificate-renewal workflow. Back up CA material, preserve trust relationships, and plan certificate changes explicitly.

## Authentication and secrets

### Redis ACLs

Define application users in `redis_security_users_acl_conf`:

```yaml
redis_security_users_acl_conf:
  - name: "{{ production_info.redis.client.username }}"
    pass: "{{ production_info.redis.client.password }}"
    enable: true
    key_pattern: "~redis:*"
    commands: "+@write +@read -@admin"
```

Additional entries support replication, monitoring, and bootstrap users. The supplied bootstrap user is named `init` and has `+@all`; restrict or remove bootstrap privileges after provisioning through a controlled process.

The sample command-renaming configuration changes `CONFIG` to `CONFIG_SECURE` and `DEBUG` to `DEBUG_SECURE`. The exporter derives its configuration command name from that mapping. Validate ACL subcommand permissions against your Redis version and exporter behavior.

### Redis password-hashing caveat

The current ACL template combines `password_hash(hashtype='sha256')` with the Redis `>` prefix. This is not Redis's native pre-hashed ACL syntax: `>` accepts a password, whereas `#` accepts a raw, lowercase, 64-character SHA-256 hexadecimal digest. Using the current branch changes the effective password to the generated hash string rather than configuring the intended original password.

Correct the template consistently for application, replication, exporter, and bootstrap users before relying on hashing. The intended token generation is equivalent to:

```jinja2
{% if user.pass == 'nopass' %}
nopass
{% elif redis_security_use_hash_password %}
#{{ user.pass | hash('sha256') }}
{% else %}
>{{ user.pass }}
{% endif %}
```

Exporter HTTP bcrypt hashes are separate from Redis ACL hashes and are not interchangeable.

### Vault schema and variable names

The secrets example defines `vm_conn_info` and `production_info`, covering connection credentials, repository credentials, Redis users, replication credentials, optional key passphrases, Sentinel fields, and exporter authentication.

Exporter settings reference `redis_sensitive_credentials`, which is not defined in the supplied files. Normalize those references to `production_info`, or deliberately define an alias in inventory:

```yaml
redis_sensitive_credentials: "{{ production_info }}"
```

Ansible Vault protects stored inventory secrets, not every generated runtime file. The exporter environment file currently contains Redis credentials and is rendered with mode `0644`; harden permissions and runtime access. Keep generated keys, certificates, Vault passwords, logs, and local environment artifacts out of version control as appropriate.

## Monitoring

The supplied inventory selects Redis Exporter version `1.86.0` for Linux `amd64` and downloads its archive from the configured artifact server. Change `redis_exporter_version`, `redis_exporter_os_arch`, and the repository URL to match available artifacts and target architecture.

Exporter environment settings live in `redis_exporter_env_params`. They include the Redis connection URI and ACL credentials, client TLS files, HTTPS server files, metrics path, Basic authentication, and optional metrics collection controls.

After correcting certificate identities and trust, a Prometheus scrape job can use:

```yaml
scrape_configs:
  - job_name: redis
    scheme: https
    metrics_path: /metrics
    static_configs:
      - targets:
          - redis-node-01.example.net:9121
          - redis-node-02.example.net:9121
    basic_auth:
      username: monitor-http
      password_file: /etc/prometheus/secrets/redis-exporter-password
    tls_config:
      ca_file: /etc/prometheus/certs/redis-exporter-trust.pem
```

Replace the example targets and username. The password file contains the original HTTP authentication password, not its bcrypt hash. The trust file must validate the exporter web certificates, which are separate from the Redis CA in the current implementation.

The supplied exporter configuration enables debug mode and sets `REDIS_EXPORTER_SKIP_TLS_VERIFICATION` to `true`. Review both before production use; provision matching certificate SANs and trusted CA material rather than retaining verification bypasses.

## Validation and operations

### Service health

On each Redis host:

```bash
sudo systemctl status redis redis-exporter
sudo journalctl -u redis -u redis-exporter --since "15 minutes ago"
```

### Cluster health

From a client with a trusted CA, valid client certificate, and an ACL user permitted to inspect the cluster:

```bash
redis-cli -c --tls \
  --cacert /secure/redis/root.crt \
  --cert /secure/redis/client.crt \
  --key /secure/redis/client.key \
  -h redis-node-01.example.net -p 6380 \
  --user cluster-admin --askpass CLUSTER INFO
```

Repeat with `CLUSTER NODES` to inspect actual primaries, replicas, and advertised addresses. Confirm that the cluster is healthy, all expected slots are assigned, and replicas are placed in appropriate failure domains.

### Metrics endpoint

```bash
curl --fail \
  --cacert /secure/monitoring/redis-exporter-trust.pem \
  --user monitor-http \
  https://redis-node-01.example.net:9121/metrics
```

`curl` prompts for the HTTP password when it is omitted. Check Redis connectivity metrics as well as HTTP availability.

### Repeated runs and scaling

Do not treat the current full playbook as a rolling-upgrade or scale-out workflow. Cluster creation has no existing-cluster guard, certificate tasks delete material, and service tasks use `state: started` rather than reliably restarting services after configuration changes.

Adding hosts to inventory alone does not safely rebalance an existing cluster. Plan node addition, replica assignment, slot migration, certificate lifecycle, and failure testing separately.

## Pre-deployment checklist

- [ ] Supply the missing `firewall` role and required Ansible collections.
- [ ] Replace controller-specific SSH, inventory, interface, certificate, and path assumptions.
- [ ] Normalize `redis_sensitive_credentials` and populate the full Vault schema.
- [ ] Define `redis_cluster_tls_ca_cert`, `redis_cluster_tls_controller_cert`, and `redis_cluster_tls_controller_key` for the controller's generated certificate files; bootstrap references these names, but the supplied inventory does not define them.
- [ ] Fix Redis ACL hash generation and hash prefixes for every user category.
- [ ] Fix cluster boolean rendering. Several template conditions test `value != ''`, which renders YAML `false` as `yes`; in particular, this can enable `cluster-replica-no-failover` and prevent the expected automatic failover.
- [ ] Reconcile Redis client ports, cluster bus listening ports, and advertised endpoints.
- [ ] Make bootstrap TLS arguments conditional for non-TLS operation, add a single-node guard, and guard against recreating an existing cluster.
- [ ] Handle noninteractive cluster-create confirmation explicitly; the current command does not include `--cluster-yes` or an equivalent confirmation input.
- [ ] Correct the exporter play's host scope and certificate SANs.
- [ ] Review certificate deletion, private-key passphrase consistency, certificate validity, and CA backup/rotation procedures.
- [ ] Restore package signature checks and HTTPS certificate verification where disabled.
- [ ] Restrict generated secret-bearing file permissions and protect logs and cached data.
- [ ] Fix service restart/reload behavior so rendered changes take effect.
- [ ] Fix OS preparation commands where necessary: the transparent-hugepage command currently uses shell redirection with `ansible.builtin.command`, which does not interpret that redirection.
- [ ] Validate all rendered Redis directives against the exact installed Redis version.
- [ ] Test cluster bootstrap, ACL authentication, mutual TLS, monitoring, and failover in a disposable environment.

## License and author

Author: Alireza Zamirian.

`galaxy.yml` declares `GPL-2.0-or-later`. Some role files retain `MIT-0` SPDX headers and scaffold README files mention BSD; reconcile these notices and provide a definitive license file before publishing a uniform licensing claim.

