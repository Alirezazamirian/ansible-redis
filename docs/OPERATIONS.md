# Operations guide

## Daily checks

```bash
sudo systemctl is-active redis redis-exporter
sudo systemctl status redis redis-exporter --no-pager
sudo journalctl -u redis -u redis-exporter --since "1 hour ago" --no-pager
```

## Cluster inspection

```bash
redis-cli -c --tls \
  --cacert /secure/redis/root.crt \
  --cert /secure/redis/client.crt \
  --key /secure/redis/client.key \
  -h redis-node-01.example.net -p 6380 \
  --user cluster-admin --askpass CLUSTER INFO
```

Check:

- `cluster_state`
- `cluster_slots_assigned`
- `cluster_slots_ok`
- `cluster_known_nodes`
- `cluster_size`
- Replica health and slot coverage

## Exporter inspection

```bash
curl --fail \
  --cacert /secure/monitoring/redis-exporter-trust.pem \
  --user monitor-http \
  [https://redis-node-01.example.net:9121/metrics](https://redis-node-01.example.net:9121/metrics)
```

## Useful Ansible commands

```bash
ansible-inventory -i inventories/production/hosts.yml --graph --ask-vault-pass

ansible-playbook -i inventories/production/hosts.yml \
  playbooks/production/all.yml --syntax-check --ask-vault-pass

ansible-playbook -i inventories/production/hosts.yml \
  playbooks/production/all.yml --ask-vault-pass --check --diff
```

Use `--check` carefully: it does not replace a full test in an isolated environment, especially for cluster bootstrap, certificate generation, and service lifecycle tasks.

## Incident response

1. Preserve logs and relevant configuration.
2. Determine whether the issue affects availability, data durability, authentication, or TLS trust.
3. Check Redis service status and cluster membership.
4. Review recent Ansible runs and configuration changes.
5. Rotate exposed credentials or certificates immediately.
6. Restore from validated backups only after assessing replication and persistence state.

