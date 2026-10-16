# Security Policy

## Supported versions

Security fixes are applied to the latest default branch. Older branches and released versions may not receive fixes.

## Reporting a vulnerability

Do not report exploitable vulnerabilities, leaked credentials, or private-key exposure in a public GitHub issue.

Contact the maintainer privately with:

- A description of the issue.
- Affected roles, playbooks, variables, or files.
- Reproduction steps.
- Potential impact.
- Suggested remediation, if available.

Please allow reasonable time for assessment and remediation before public disclosure.

## Handling sensitive material

Never commit:

- Ansible Vault passwords or password files.
- Plaintext `vault.yml` files.
- TLS private keys or certificates.
- Redis ACL passwords.
- Repository credentials.
- SSH keys or connection passwords.
- Unredacted logs containing credentials.

If sensitive material is committed, rotate every exposed credential and certificate immediately. Removing a file from Git history does not invalidate exposure.

