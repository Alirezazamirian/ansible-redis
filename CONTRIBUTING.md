# Contributing

Thank you for considering a contribution to this Ansible Redis Cluster project.

## Before opening an issue

Please include:

- The branch or commit you are using.
- Your operating system and Ansible version.
- The target distribution and Redis version.
- The inventory group and relevant variables involved.
- The exact command you ran.
- Redacted logs or task output.

Never paste private keys, certificates, Vault contents, passwords, IP addresses, or other sensitive values.

## Reporting bugs

Open an issue with:

1. A short summary.
2. Expected behavior.
3. Actual behavior.
4. Minimal reproduction steps.
5. Redacted relevant inventory variables.
6. Relevant Ansible output, using `-vvv` only when necessary.

## Suggesting enhancements

Describe the problem first, then the proposed solution. Include the intended use case, affected role, and any compatibility or security implications.

## Submitting changes

1. Fork the repository.
2. Create a feature branch:
   ```bash
   git checkout -b feature/short-description
   ```
3. Make focused changes.
4. Validate syntax:
   ```bash
   ansible-playbook -i inventories/production/hosts.yml \
     playbooks/production/all.yml --syntax-check --ask-vault-pass
   ```
5. Run Ansible linting where available:
   ```bash
   ansible-lint
   ```
6. Test changes in an isolated environment before opening a pull request.
7. Submit a pull request with a clear description and references to related issues.

## Code style

- Use YAML with two-space indentation.
- Prefer fully qualified collection names, for example `ansible.builtin.template`.
- Add `no_log: true` to tasks that handle credentials or generated private material.
- Avoid committing generated certificates, private keys, Vault files, logs, or local cache data.
- Keep role defaults empty when values belong in inventory.
- Document new variables in the README and provide safe examples.

## Security

Do not open public issues for exploitable vulnerabilities or leaked credentials. Contact the maintainer privately, or use a responsible-disclosure channel if one is published.

## License

By contributing, you agree that your contributions are licensed under the repository license, GPL-2.0-or-later.

