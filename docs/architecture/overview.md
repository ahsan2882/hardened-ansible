# Architecture Overview

## Introduction

Hardened Ansible is an infrastructure automation project that provisions and hardens Ubuntu-based web servers using Ansible.

The project establishes a reusable server baseline, configures SSH access and firewall rules, deploys Nginx, enables Fail2ban, and provisions HTTPS using certificates issued by a locally managed Certificate Authority (CA).

The current example inventory contains two web servers:

- `web01`
- `web02`

The inventory is provided as a template. Actual host addresses and connection settings belong in the local inventory, which is excluded from version control.

## Architecture

The project uses an Ansible control node to manage remote web servers over SSH.

The control node is responsible for:

- Executing Ansible playbooks.
- Managing inventory and configuration.
- Generating and maintaining the local CA.
- Collecting server certificate signing requests (CSRs).
- Signing CSRs using the local CA.
- Deploying signed certificates and the CA certificate.

Each managed web server is responsible for:

- Maintaining its own TLS private key.
- Generating its certificate signing request.
- Running the Nginx web server.
- Enforcing the configured SSH and firewall policies.
- Running Fail2ban to protect SSH access.

The CA private key remains on the control node. Server private keys are generated and retained on their respective web servers.

## Project Structure

```text
hardened-ansible/
├── ansible.cfg
├── requirements.yml
├── inventory/
│   ├── hosts.example.yml
│   └── group_vars/
│       ├── all.yml
│       └── webservers.yml
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
└── docs/
    ├── architecture/
    ├── security/
    └── testing/
```

Each role separates configurable defaults, implementation tasks, and any required handlers or templates. Every role also has its own Molecule test scenario.

## Configuration Management

The root-level `ansible.cfg` defines the project's default inventory, role search path, collection path, SSH host-key checking, and connection settings.

Configuration is organized into:
- inventory/group_vars/all.yml: shared settings such as timezone and base packages.
- inventory/group_vars/webservers.yml: settings specific to managed web servers.
- Role defaults: configurable behavior and role-specific values.
- inventory/hosts.yml: local inventory containing actual host connection details.

The committed inventory/hosts.example.yml provides an example inventory using documentation-only IP addresses. Copy it to inventory/hosts.yml and adjust it for the target environment.

## Deployment Workflow

The main deployment playbook is playbooks/site.yml.

It applies the following roles to the webservers group:
1. common: installs base packages and configures the system timezone.
2. users: creates and configures managed user accounts and sudo policies.
3. ssh_hardening: applies the SSH security configuration.
4. firewall: configures UFW policies and permitted inbound ports.
5. fail2ban: installs and configures the SSH jail.
6. nginx: installs Nginx and deploys the landing page.
7. TLS deployment tasks: deploy certificates, configure HTTPS, and redirect HTTP traffic.

The playbook uses privilege escalation for server configuration.

## Certificate Lifecycle

TLS provisioning is handled through separate playbooks and the tls and local_ca roles.

The intended lifecycle is:
1. Generate a server private key and CSR on each web server.
2. Fetch the CSR files to the Ansible control node.
3. Create the local CA and its self-signed certificate.
4. Sign the collected server CSRs using the local CA.
5. Deploy each signed certificate and the CA certificate to the corresponding web server.
6. Update the server's trusted CA store.
7. Configure the Nginx HTTPS virtual host and HTTP-to-HTTPS redirect.

Server private keys are not transferred to the control node. The CA private key is stored locally in the project's PKI workspace and must be protected.

See [Certificate Lifecycle](certificate-lifestyle.md) for further details.

## Testing

The project uses Molecule to test individual roles in isolated containers.

The security validation playbook in tests/validation/security.yml checks the deployed server configuration, service status, SSH settings, certificate trust, and HTTP/HTTPS behavior.

See the testing documentation for details about the test strategy and execution commands.
Security Considerations
- Use SSH public-key authentication for Ansible connections.
- Keep the CA private key outside version control and restrict access to it.
- Do not commit local inventories, private keys, generated certificates, or other PKI artifacts.
- Review firewall rules before applying them to remote systems.
- Ensure the Ansible control node has secure access to the managed servers.
- Test changes in an isolated environment before applying them to production systems.
