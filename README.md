# Hardened Ansible

[![Ansible CI](https://github.com/ahsan2882/hardened-ansible/actions/workflows/ci.yml/badge.svg?branch=main)](https://github.com/ahsan2882/hardened-ansible/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/github/license/ahsan2882/hardened-ansible)](LICENSE.md)

A reusable Ansible project for configuring and hardening Ubuntu-based web servers. It provides a modular server baseline, SSH access controls, UFW firewall rules, Fail2ban, Nginx, and TLS certificates issued by a locally managed Certificate Authority (CA).

The project is designed around a dedicated Ansible control node and one or more managed web servers. It includes eight reusable roles, role-level Molecule tests, a security-validation playbook, and a GitHub Actions CI workflow.

## Features

- **System baseline:** Package installation and configurable timezone management.
- **User management:** Dedicated automation account, SSH authorized keys, and validated sudo policy.
- **SSH hardening:** Key-based access, restricted users, reduced authentication attempts, and hardened forwarding/session settings.
- **Firewall:** UFW default-deny inbound policy, explicit service rules, and rate-limited SSH.
- **Brute-force protection:** Fail2ban SSH jail using UFW as its ban action.
- **Web server:** Nginx virtual hosts, managed landing page, hidden-file restrictions, security headers, and per-client request rate limiting.
- **TLS/PKI:** Server-side private-key and CSR generation, local CA management, CSR validation, certificate signing, and HTTPS deployment.
- **Automated testing:** Eight independent Molecule scenarios using Podman and Ubuntu 24.04.
- **CI:** GitHub Actions syntax checks and a matrix job that runs every role's Molecule scenario.

## Architecture

The control node runs Ansible, stores the local CA material, collects server CSRs, and signs validated requests. Managed Ubuntu servers are configured over SSH and retain their own TLS private keys. Nginx serves HTTP/HTTPS traffic; the local CA certificate must be trusted by clients that validate the server certificates.

See [Architecture Overview](docs/architecture/overview.md), [Network Topology](docs/architecture/network-topology.md), and [Certificate Lifecycle](docs/architecture/certificate-lifecycle.md).

## Roles

| Role            | Responsibility                                                     |
| --------------- | ------------------------------------------------------------------ |
| `common`        | Base packages, APT cache management, and timezone                  |
| `users`         | Automation accounts, authorized keys, and sudo policy              |
| `ssh_hardening` | OpenSSH access and session hardening                               |
| `firewall`      | UFW defaults, inbound rules, logging, and SSH rate limiting        |
| `fail2ban`      | SSH jail and UFW ban action                                        |
| `nginx`         | Nginx installation, HTTP site, security headers, and rate limiting |
| `local_ca`      | Local CA creation and validation/signing of server CSRs            |
| `tls`           | Server key/CSR generation and signed-certificate deployment        |

## Repository Layout

```text
.
├── .github/workflows/ci.yml
├── docs/
│   ├── architecture/
│   ├── security/
│   └── testing/
├── inventory/
│   ├── group_vars/
│   └── hosts.example.yml
├── playbooks/
│   ├── site.yml
│   ├── firewall.yml
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

For local development and role testing:

- Python and `venv`
- Ansible Core (CI currently pins `2.21.5`)
- Ansible collections listed in `requirements.yml`
- Podman
- Molecule and the Molecule Podman driver
- SSH access to the managed Ubuntu hosts for deployment

The GitHub Actions workflow currently uses Python 3.12, Ansible Core 2.21.5, Molecule 26.10.0, and Podman.

## Getting Started

### 1. Clone the repository

```bash
git clone https://github.com/ahsan2882/hardened-ansible.git
cd hardened-ansible
```

### 2. Create a virtual environment

```bash
python3 -m venv .venv
source .venv/bin/activate
```

### 3. Install Ansible and Molecule

```bash
python -m pip install --upgrade pip
pip install ansible-core==2.21.5 molecule==26.10.0 'molecule-plugins[podman]'
```

### 4. Install Ansible collections

```bash
ansible-galaxy collection install -r requirements.yml -p ./collections
```

### 5. Configure the inventory

```bash
cp inventory/hosts.example.yml inventory/hosts.yml
```

Edit `inventory/hosts.yml` with the target host addresses, SSH connection settings, and TLS identities. The local inventory is ignored by Git. Keep real host details and credentials out of committed files.

Review `inventory/group_vars/all.yml` and `inventory/group_vars/webservers.yml` as well. In particular, configure the SSH public key in `users_ansible_authorized_keys` before applying the user and SSH-hardening roles.

### 6. Check inventory and connectivity

```bash
ansible-inventory --list
ansible webservers -m ansible.builtin.ping
```

Run Ansible commands from the repository root so that the project `ansible.cfg` is loaded.

### 7. Prepare TLS material for the initial deployment

The `site.yml` playbook also deploys TLS artifacts, so the CA certificate and signed server certificates must exist first. For a new environment, run:

```bash
ansible-playbook playbooks/generate-csrs.yml
ansible-playbook playbooks/local-ca.yml
ansible-playbook playbooks/sign-csrs.yml
```

Review the generated CSR identities and SANs before signing. Protect the CA private key and do not include the `pki/` directory in source archives or version control.

### 8. Deploy the baseline

```bash
ansible-playbook playbooks/site.yml
```

Run this only after the inventory, SSH access, and TLS material are ready. The playbook configures the baseline roles and then deploys the signed certificates and Nginx TLS configuration.

## Playbooks and Deployment

| Playbook                      | Purpose                                                                                         |
| ----------------------------- | ----------------------------------------------------------------------------------------------- |
| `playbooks/site.yml`          | Applies the common, users, SSH, firewall, Fail2ban, and Nginx roles, then deploys TLS artifacts |
| `playbooks/firewall.yml`      | Applies the firewall role independently                                                         |
| `playbooks/generate-csrs.yml` | Generates server private keys and CSRs on managed hosts, then fetches the CSRs                  |
| `playbooks/local-ca.yml`      | Creates the local CA                                                                            |
| `playbooks/sign-csrs.yml`     | Validates all collected CSRs, then signs them                                                   |
| `playbooks/deploy-tls.yml`    | Deploys signed certificates and CA trust, and configures HTTPS/HTTP redirection                 |

**Deployment warning:** Review the inventory, SSH access, firewall policy, and role variables before applying playbooks to remote systems. Test changes against disposable environments first. Do not run deployment playbooks against production without an explicit change plan and recovery access.

## TLS Certificate Workflow

The certificate workflow is deliberately split into distinct operations:

1. `generate-csrs.yml` creates each server's private key and CSR on that server, then fetches the CSR to the control node.
2. `local-ca.yml` initializes the local CA if required.
3. `sign-csrs.yml` validates the signature, Common Name, SANs, critical `CA:FALSE` basic constraint, and server-authentication EKU for every CSR before signing.
4. `deploy-tls.yml` deploys the signed certificate and CA certificate, updates the system trust store when needed, and configures Nginx HTTPS and HTTP redirection.

The server private keys remain on their respective managed hosts. The CA private key is highly sensitive and must be protected and backed up separately from the repository. The `pki/` directory is excluded from Git; do not add private keys, certificates, CSRs, or other PKI artifacts to version control.

See [Certificate Lifecycle](docs/architecture/certificate-lifecycle.md) and [TLS and PKI](docs/security/tls-and-pki.md).

## Testing

### Run one role's Molecule scenario

```bash
cd roles/nginx
molecule test
```

Repeat from another role directory to test that role.

### Run all eight role scenarios

From the repository root, with the virtual environment active:

```bash
for role_dir in roles/*; do
    if [ -f "$role_dir/molecule/default/molecule.yml" ]; then
        role="${role_dir##*/}"
        echo "Testing $role"
        (
            cd "$role_dir" &&
            molecule test
        ) || exit 1
    fi
done
```

### Run syntax checks

```bash
for playbook in \
    playbooks/site.yml \
    playbooks/generate-csrs.yml \
    playbooks/local-ca.yml \
    playbooks/sign-csrs.yml \
    playbooks/deploy-tls.yml \
    tests/validation/security.yml
do
    ansible-playbook --syntax-check -i inventory/hosts.example.yml "$playbook" || exit 1
done
```

### Validate configured servers

```bash
ansible-playbook tests/validation/security.yml
```

This playbook checks selected live controls, including Nginx and Fail2ban service status, UFW activation, effective SSH settings, certificate trust, HTTPS response, and HTTP redirection. It requires the managed servers to be configured and reachable.

The project includes GitHub Actions CI for syntax checks and all eight Molecule role scenarios. CI does not provision or deploy to the project's real managed servers. The `tests/integration/` directory is a placeholder; a separate automated multi-role VM integration suite has not yet been implemented.

See [Testing Strategy](docs/testing/testing-strategy.md), [Molecule Testing](docs/testing/molecule.md), and [Integration Testing](docs/testing/integration-testing.md).

## Security Scope

This project provides a configurable baseline, not a guarantee of complete server security. It does not currently provide automated certificate renewal/revocation, centralized logging, OS patch orchestration, MFA, vulnerability scanning, or application-level security testing. Review the [Hardening Baseline](docs/security/hardening-baseline.md) and the individual security guides before adapting it to a production environment.

## Documentation

| Document                                                            | Description                                           |
| ------------------------------------------------------------------- | ----------------------------------------------------- |
| [Architecture Overview](docs/architecture/overview.md)              | Components, responsibilities, and deployment sequence |
| [Network Topology](docs/architecture/network-topology.md)           | Management and client traffic flows                   |
| [Certificate Lifecycle](docs/architecture/certificate-lifecycle.md) | CSR generation, CA creation, signing, and deployment  |
| [Hardening Baseline](docs/security/hardening-baseline.md)           | Implemented security controls and limitations         |
| [SSH Security](docs/security/ssh-security.md)                       | OpenSSH hardening                                     |
| [Firewall Policy](docs/security/firewall-policy.md)                 | UFW policy and allowed ports                          |
| [TLS and PKI](docs/security/tls-and-pki.md)                         | CA, CSR validation, certificates, and Nginx TLS       |
| [Testing Strategy](docs/testing/testing-strategy.md)                | Test layers and CI                                    |
| [Molecule Testing](docs/testing/molecule.md)                        | Role-level container tests                            |
| [Integration Testing](docs/testing/integration-testing.md)          | Current integration-test status and future approach   |

## License

MIT License. See [LICENSE.md](LICENSE.md).
