# Hardened Ansible

A reusable Ansible infrastructure automation project for deploying and hardening Ubuntu-based web servers.

The project uses modular Ansible roles to configure system packages, user access, SSH security, firewall policies, Fail2ban, Nginx, and TLS certificates through a locally managed Certificate Authority.

## Features

- **Modular architecture:** Eight reusable Ansible roles.
- **System configuration:** Package management and timezone configuration.
- **User management:** Dedicated automation account and sudo policies.
- **SSH hardening:** Public-key authentication and restricted access.
- **Firewall:** UFW with explicit inbound, outbound, and routed policies.
- **Intrusion protection:** Fail2ban SSH jail.
- **Web server:** Nginx installation and configuration.
- **TLS/PKI:** Local Certificate Authority, server-generated keys and CSRs, and signed certificates.
- **Role testing:** Molecule scenarios using Podman and Ubuntu 24.04.
- **Security validation:** Automated checks for selected security controls.

## Architecture

The project follows a control-node and managed-node architecture.

- The control node runs Ansible and manages the inventory, playbooks, roles, and local Certificate Authority.
- Managed nodes are Ubuntu-based web servers configured through SSH.
- Nginx provides HTTP and HTTPS services.
- A local CA issues certificates for the managed web servers.

See the [Architecture Overview](docs/architecture/overview.md) and [Network Topology](docs/architecture/network-topology.md) for details.

## Ansible Roles

| Role            | Responsibility                                      |
| --------------- | --------------------------------------------------- |
| `common`        | Base packages and timezone configuration            |
| `users`         | User accounts and sudo policies                     |
| `ssh_hardening` | SSH authentication and access restrictions          |
| `firewall`      | UFW firewall policies                               |
| `fail2ban`      | SSH brute-force protection                          |
| `nginx`         | Nginx installation and configuration                |
| `local_ca`      | Local Certificate Authority and certificate signing |
| `tls`           | Server keys, CSRs, and TLS deployment               |

## Repository Structure

```text
.
├── docs/
│   ├── architecture/
│   ├── security/
│   └── testing/
├── inventory/
│   ├── group_vars/
│   └── hosts.example.yml
├── playbooks/
│   ├── site.yml
│   ├── generate-csrs.yml
│   ├── local-ca.yml
│   ├── sign-csrs.yml
│   └── deploy-tls.yml
├── roles/
│   ├── common/
│   ├── users/
│   ├── ssh_hardening/
│   ├── firewall/
│   ├── fail2ban/
│   ├── nginx/
│   ├── local_ca/
│   └── tls/
├── tests/
│   ├── integration/
│   └── validation/
├── ansible.cfg
├── requirements.yml
└── README.md
```

## Requirements

- Ansible Core
- Python virtual environment
- Required Ansible collections
- SSH access to managed Ubuntu hosts
- Podman and Molecule for role testing

## Getting Started

### 1. Clone the repository

```bash
git clone git@github.com:ahsan2882/hardened-ansible.git
cd hardened-ansible
```

### 2. Create and activate a virtual environment

```bash
python3 -m venv .venv
source .venv/bin/activate
```

### 3. Install Ansible and Molecule

```bash
pip install ansible-core molecule 'molecule-plugins[podman]'
```

### 4. Install required collections

```bash
ansible-galaxy collection install -r requirements.yml
```

### 5. Configure the inventory

Copy the example inventory:

```bash
cp inventory/hosts.example.yml inventory/hosts.yml
```

Update `inventory/hosts.yml` with your managed hosts, SSH connection details, and certificate identities.

The local inventory is excluded from Git to prevent environment-specific configuration from being published.

### 6. Verify connectivity

```bash
ansible-inventory --list
ansible webservers -m ansible.builtin.ping
```

### 7. Deploy the baseline

```bash
ansible-playbook playbooks/site.yml
```

Run Ansible commands from the repository root so that the project-level `ansible.cfg` is used.

## TLS Certificate Workflow

The TLS workflow separates server-side key generation from CA operations.

The general process is:
1. Generate server private keys and CSRs on managed hosts.
2. Fetch CSRs to the control node.
3. Create the local CA.
4. Sign the server CSRs.
5. Deploy signed certificates and the CA certificate.
6. Configure Nginx for HTTPS and HTTP redirection.

Refer to [Certificate Lifecycle](docs/architecture/certificate-lifecycle.md) and [TLS and PKI](docs/security/tls-and-pki.md).

Security: The CA private key is sensitive and must never be committed to Git. The pki/ directory is excluded from version control.

## Testing

### Molecule role tests

Each of the eight roles has an independent Molecule scenario using Podman and Ubuntu 24.04.

```bash
cd roles/common
molecule test
```

Run the same command from another role directory to test that role.

### Security validation

From the repository root:

```bash
ansible-playbook tests/validation/security.yml
```

This validates selected SSH, firewall, Fail2ban, Nginx, and TLS settings on the managed hosts.

See the [Testing Strategy](docs/testing/testing-strategy.md) and [Molecule Testing](docs/testing/molecule.md).

## Security Considerations

- Use SSH public-key authentication.
- Protect the automation account's private key.
- Keep the CA private key secure and backed up separately.
- Review inventory and role variables before deployment.
- Test configuration changes in disposable environments first.
- Do not commit private keys, certificates, credentials, or production inventory.

## Documentation

| Document                                                            | Description                         |
| ------------------------------------------------------------------- | ----------------------------------- |
| [Architecture Overview](docs/architecture/overview.md)              | Project architecture and components |
| [Network Topology](docs/architecture/network-topology.md)           | Hosts, services, and network flows  |
| [Certificate Lifecycle](docs/architecture/certificate-lifecycle.md) | TLS certificate workflow            |
| [Hardening Baseline](docs/security/hardening-baseline.md)           | Implemented security controls       |
| [SSH Security](docs/security/ssh-security.md)                       | SSH hardening configuration         |
| [Firewall Policy](docs/security/firewall-policy.md)                 | UFW configuration                   |
| [TLS and PKI](docs/security/tls-and-pki.md)                         | Certificate Authority and TLS       |
| [Testing Strategy](docs/testing/testing-strategy.md)                | Testing approach and coverage       |
| [Molecule Testing](docs/testing/molecule.md)                        | Role-level testing                  |
| [Integration Testing](docs/testing/integration-testing.md)          | Integration testing approach        |

## Current Scope

This project currently provides reusable role-level automation, a complete deployment playbook, TLS certificate management, and security validation.

A dedicated automated integration test suite and CI pipeline are not yet implemented.

## License

MIT License

Copyright (c) 2026 Ahsan Shahid
