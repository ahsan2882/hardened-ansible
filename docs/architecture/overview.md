# Architecture Overview

## Introduction

Hardened Ansible provisions and hardens Ubuntu-based web servers using reusable Ansible roles. The baseline configures system packages, automation access, SSH, UFW, Fail2ban, Nginx, and TLS through a locally managed Certificate Authority (CA).

The example inventory describes two web servers, `web01` and `web02`, using documentation-only addresses. Replace these with environment-specific values in the ignored local inventory.

## Components and Responsibilities

### Ansible control node

The control node:

- Runs Ansible playbooks and role tests.
- Stores inventory and project configuration.
- Generates and stores the local CA private key and certificate.
- Receives CSRs from managed servers.
- Validates and signs server CSRs.
- Deploys signed certificates and the CA certificate to managed servers.

### Managed web servers

Each managed server:

- Is accessed over SSH using the configured automation account.
- Runs the common, users, SSH, firewall, Fail2ban, and Nginx roles.
- Generates and retains its own TLS private key.
- Generates a CSR containing its configured identity and SANs.
- Serves the managed site over HTTPS after certificate deployment.

Server private keys are not fetched to the control node. The CA private key remains on the control node and requires separate protection.

## Project Structure

```text
hardened-ansible/
├── .github/workflows/ci.yml
├── ansible.cfg
├── requirements.yml
├── inventory/
│   ├── hosts.example.yml
│   └── group_vars/
│       ├── all.yml
│       └── webservers.yml
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
└── docs/
```

Each role uses defaults and tasks, with handlers and templates where needed. Each of the eight roles has a dedicated Molecule default scenario.

## Configuration

The project-level `ansible.cfg` configures:

- `inventory/hosts.yml` as the default inventory.
- `roles/` as the role search path.
- `./collections` as the Ansible collection path.
- SSH host-key checking.
- SSH pipelining.

Configuration is divided between:

- `inventory/group_vars/all.yml`: shared settings such as timezone, SSH port, package list, and server certificate validity.
- `inventory/group_vars/webservers.yml`: web-server connection settings, TLS paths, and the automation account's authorized public key.
- Role defaults: role-specific behavior and security settings.
- `inventory/hosts.yml`: local host addresses and per-host certificate identity.

Copy `inventory/hosts.example.yml` to `inventory/hosts.yml` and customize it. Do not commit the local inventory or credentials.

## Main Deployment

`playbooks/site.yml` applies two plays to the `webservers` group:

1. The baseline play applies `common`, `users`, `ssh_hardening`, `firewall`, `fail2ban`, and `nginx`.
2. The TLS play imports the `tls` role's `deploy` task file to install certificates and configure HTTPS.

The order is intentional: the Nginx role installs Nginx and its global security configuration before the TLS tasks add the HTTPS virtual host. The global Nginx configuration defines the request-rate-limit zone used by the HTTP and HTTPS sites.

`playbooks/firewall.yml` can apply the firewall role independently.

## Certificate Operations

Certificate provisioning is split into separate playbooks:

| Playbook            | Operation                                                               |
| ------------------- | ----------------------------------------------------------------------- |
| `generate-csrs.yml` | Generate server private keys and CSRs on managed hosts, then fetch CSRs |
| `local-ca.yml`      | Create the local CA                                                     |
| `sign-csrs.yml`     | Inspect and validate all CSRs, then sign them                           |
| `deploy-tls.yml`    | Install certificates and configure Nginx TLS                            |

See [Certificate Lifecycle](certificate-lifecycle.md) for the detailed sequence.

## Testing and CI

Each role has a Molecule scenario using Podman and a systemd-capable Ubuntu 24.04 container. GitHub Actions runs playbook syntax checks and a matrix of all eight role scenarios. The separate `tests/validation/security.yml` playbook validates selected settings on already-configured managed servers.

The `tests/integration/` directory is currently a placeholder; there is no automated end-to-end VM integration suite.

## Security Boundaries

- The control node holds the CA private key; protect it independently of the repository.
- Managed nodes retain their own server private keys.
- SSH access depends on the configured public key and the automation account.
- UFW controls host-level network access; it does not replace cloud security groups or network ACLs.
- CI tests syntax and isolated roles; it does not deploy to real infrastructure.
