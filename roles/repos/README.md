# Ansible Role: repos

## Description
This Ansible role is designed to configure and manage local YUM repositories on RedHat-based Linux distributions such as Rocky Linux. It ensures that the system is using locally available YUM repositories for package management, improving reliability and speed.

## Project Structure
```
repos
├── files
├── README.md
├── defaults
│   └── main.yml
├── handlers
│   └── main.yml
├── meta
│   └── main.yml
├── tasks
│   └── main.yml
├── templates
│   └── yum
│       └── local
│           ├── *.repo.j2 (Various repository template files)
├── tests
│   ├── inventory
│   └── test.yml
└── vars
    └── main.yml
```

## Requirements
- Ansible 2.1 or higher
- RedHat-based distributions (RHEL, Rocky Linux, CentOS, etc.)
- Local YUM repository server available at `repos_info.mirror.address`

## Role Variables

### Defaults (`defaults/main.yml`)
```yaml
repos_info.mirror.address: "192.168.19.77"
```
This variable defines the IP address or hostname of the local YUM repository server.

### Variables (`vars/main.yml`)
Currently, no additional variables are defined in this file.

## Handlers (`handlers/main.yml`)
Currently empty. Can be used to add handlers if needed in the future.

## Tasks (`tasks/main.yml`)
This role performs the following tasks:
1. Includes `modules_deps/dnf.yml` if the OS is RedHat-based.
2. Configures local YUM repositories by rendering templates.
3. Updates the DNF cache to recognize the newly added repositories.

## Templates (`templates/yum/local/*.repo.j2`)
This directory contains Jinja2 template files for various local YUM repositories, such as:
- `local_repo_appstream.repo.j2`
- `local_repo_baseos.repo.j2`
- `local_repo_docker-ce-stable.repo.j2`
- `local_repo_elasticsearch-*.repo.j2`
- `local_repo_mongodb-org-6.0.repo.j2`
- `local_repo_nginx.repo.j2`
- `local_repo_zabbix.repo.j2`
- `offline-packages.repo.j2`

Each template is dynamically populated with the `repos_info.mirror.address` variable to set the appropriate `baseurl`.

## Dependencies (`meta/main.yml`)
No external role dependencies are required for this role.

## Usage
### Example Playbook
```yaml
- hosts: all
  become: yes
  roles:
    - repos
```

### Running the Role
Execute the role using:
```sh
ansible-playbook -i inventory playbook.yml
```

