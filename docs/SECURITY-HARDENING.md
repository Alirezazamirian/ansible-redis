# Security hardening

This document lists security decisions to review before production use.

## Secrets

- Store all credentials in an encrypted Ansible Vault file.
- Use a dedicated Vault password file or secret manager; never commit it.
- Rotate credentials after any suspected exposure.
- Restrict permissions on local secret files, for example `chmod 600`.
- Avoid embedding credentials in generated repository URLs where possible.

## Certificates

- Protect the Redis root CA private key.
- Use distinct certificates for Redis nodes, controller bootstrap, application clients, and monitoring clients.
- Define certificate SANs that match the names and addresses clients actually use.
- Plan CA rotation and certificate renewal.
- Do not rely on TLS verification bypasses in production.

## Redis ACLs

- Use least-privilege ACL users for applications and monitoring.
- Avoid broad `+@all` permissions outside a controlled bootstrap process.
- Restrict key patterns to application namespaces.
- Disable or rename dangerous commands only after validating application compatibility.
- Review ACL command permissions against the installed Redis version.

## Network

- Restrict Redis client, cluster bus, and exporter ports to required networks.
- Use firewall rules and security groups where available.
- Prefer private network paths for replication and cluster traffic.
- Monitor unexpected client connections and configuration changes.

## Repository configuration

- Prefer signed package repositories and verified HTTPS endpoints.
- Do not globally disable GPG or TLS verification in production.
- Pin package versions where reproducibility matters.
- Review mirror credentials and URL encoding behavior.

## Runtime files

- Restrict permissions on files containing Redis or exporter credentials.
- Ensure logs do not contain passwords, tokens, private keys, or full command arguments.
- Protect Ansible fact caches, which may contain sensitive host data.

