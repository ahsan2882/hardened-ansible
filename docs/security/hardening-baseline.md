# Security Hardening Baseline

## Overview

Hardened Ansible applies a reusable security baseline to Ubuntu-based web servers.

The baseline covers:

- Dedicated automation accounts and sudo policies.
- SSH access restrictions.
- Host-based firewall configuration.
- SSH brute-force protection.
- Nginx web server configuration.
- TLS encryption using a locally managed Certificate Authority.

The configuration is implemented through reusable Ansible roles.

## Security Controls

| Role | Responsibility |
|---|---|
| common | Installs required system packages and configures timezone |
| users | Manages user accounts and sudo policies |
| ssh_hardening | Restricts SSH authentication and access |
| firewall | Configures UFW inbound, outbound, and routed policies |
| fail2ban | Protects SSH against repeated authentication failures |
| nginx | Deploys and configures the Nginx web server |
| local_ca | Creates and manages the local Certificate Authority |
| tls | Generates server keys and CSRs and deploys signed certificates |

## User and Privilege Management

The `users` role creates a dedicated `ansible` account.

The default configuration:

- Creates a home directory.
- Uses `/bin/bash` as the login shell.
- Adds the account to the `sudo` group.
- Configures passwordless sudo.
- Validates sudoers files using `visudo` before installing them.
- Removes the default cloud-init sudo policy file.

Passwordless sudo is configured for automation convenience. Access to the Ansible account and its SSH private key must therefore be tightly controlled.

## SSH Hardening

The `ssh_hardening` role:

- Disables root login.
- Disables password authentication.
- Enables public-key authentication.
- Disables keyboard-interactive authentication.
- Limits authentication attempts to three.
- Restricts access to explicitly configured users.
- Validates the SSH configuration before deploying it.

See [SSH Security](ssh-security.md).

## Firewall Policy

The `firewall` role configures UFW with:

- Deny incoming traffic by default.
- Allow outgoing traffic by default.
- Deny routed traffic by default.
- Allow inbound TCP ports 22, 80, and 443.
- Enable low-level firewall logging.

See [Firewall Policy](firewall-policy.md).

## Brute-Force Protection

The `fail2ban` role configures an SSH jail.

Default settings:

| Setting | Value |
|---|---|
| Maximum retries | 5 |
| Find time | 10 minutes |
| Ban time | 10 minutes |
| SSH port | 22 |
| Backend | systemd |

Loopback addresses are excluded from banning.

## Web Server Security

The `nginx` role:

- Removes the default Nginx site.
- Deploys a managed virtual host.
- Uses a dedicated document root.
- Denies requests targeting hidden files.
- Validates Nginx configuration before considering deployment complete.

The TLS deployment configures HTTPS and redirects HTTP requests to HTTPS.

## Transport Security

The `tls` role configures:

- RSA 3072-bit server private keys.
- SHA-256 certificate signing requests.
- TLS 1.2 and TLS 1.3.
- Server certificates signed by the local CA.
- HTTP-to-HTTPS redirects.
- A trusted CA certificate in the system certificate store.

See [TLS and PKI](tls-and-pki.md).

## Validation

The project includes a security validation playbook:

```bash
ansible-playbook tests/validation/security.yml
```

It checks:
- Nginx configuration and service status.
- Fail2ban service and SSH jail status.
- UFW activation.
- Effective SSH hardening settings.
- Server certificate validation against the local CA.
- Successful HTTPS responses.
- HTTP-to-HTTPS redirects.

## Security Scope and Limitations

This project establishes a baseline; it does not guarantee that a server is secure against every threat.

The current configuration does not, by itself, provide:
- Automatic operating-system security upgrades.
- Centralized logging or security monitoring.
- Intrusion detection beyond the configured Fail2ban SSH jail.
- Automated certificate renewal and rotation.
- Protection against compromised control-node credentials.
- Application-level security testing.
- External exposure or vulnerability scanning.

Production deployments should review these areas according to their threat model and operational requirements.
