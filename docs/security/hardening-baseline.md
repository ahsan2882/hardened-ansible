# Security Hardening Baseline

## Overview

The project applies a configurable baseline to Ubuntu web servers through eight Ansible roles. The baseline covers system configuration, automation access, SSH, host firewall policy, SSH brute-force mitigation, Nginx, and TLS.

## Role Coverage

| Role            | Implemented responsibility                                                |
| --------------- | ------------------------------------------------------------------------- |
| `common`        | APT cache management, base packages, and timezone                         |
| `users`         | Automation account, authorized keys, and sudoers policy                   |
| `ssh_hardening` | OpenSSH authentication, access, forwarding, and session settings          |
| `firewall`      | UFW default policies, allowed services, logging, and SSH rate limiting    |
| `fail2ban`      | SSH jail using the systemd backend and UFW ban action                     |
| `nginx`         | Nginx site, global security configuration, headers, and rate limiting     |
| `local_ca`      | Local CA key/certificate creation and CSR validation/signing              |
| `tls`           | Server private keys and CSRs; signed certificate and Nginx TLS deployment |

## User and Privilege Management

The `users` role manages the `ansible` automation account by default. It:

- Creates the account and home directory.
- Adds configured supplementary groups without replacing existing groups.
- Installs configured SSH authorized keys.
- Validates managed sudoers files with `visudo`.
- Supports configurable passwordless sudo.

The role asserts that at least one automation public key is configured before continuing. Passwordless sudo is enabled by default for automation convenience; tightly protect the account and its private key.

## SSH Hardening

The `ssh_hardening` role deploys `/etc/ssh/sshd_config.d/00-ansible-hardening.conf` and validates the file with `sshd -t` before applying it. Defaults include:

- Root login disabled.
- Password and keyboard-interactive authentication disabled.
- Public-key authentication enabled.
- `AllowUsers` restricted to the configured automation account.
- Maximum authentication attempts set to three.
- X11, agent, TCP, and tunnel forwarding disabled.
- Empty passwords and user-controlled environment settings disabled.
- Login grace time and idle-session controls configured.

The role also removes an obsolete project drop-in and removes a conflicting cloud-init `AllowUsers` directive when that cloud-init file exists. Review the resulting effective configuration before deployment.

See [SSH Security](ssh-security.md).

## Firewall

UFW defaults to deny incoming, allow outgoing, and deny routed traffic. The configured SSH port is rate-limited; HTTP and HTTPS are allowed by default. Logging defaults to `low`.

See [Firewall Policy](firewall-policy.md).

## Fail2ban

The SSH jail defaults to:

| Setting           | Default                |
| ----------------- | ---------------------- |
| Enabled jail      | `sshd`                 |
| Backend           | `systemd`              |
| Ban action        | `ufw`                  |
| Maximum retries   | 5                      |
| Find time         | 10 minutes             |
| Ban time          | 10 minutes             |
| SSH port          | `system_ssh_port`      |
| Ignored addresses | IPv4 and IPv6 loopback |

The Fail2ban action uses UFW so bans integrate with the host firewall.

## Nginx

The Nginx role:

- Installs and enables Nginx.
- Removes the default enabled site by default.
- Deploys a managed landing page and virtual host.
- Disables server version tokens.
- Configures a per-client request-rate zone (default `10r/s`) and burst (default 20).
- Adds `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`, and `Permissions-Policy` headers.
- Denies access to hidden files in the managed site configuration.
- Runs `nginx -t` before reloading after managed configuration changes.

The TLS virtual host also applies the configured security headers and rate limit.

## TLS and PKI

Defaults include RSA 3072-bit server keys, RSA 4096-bit CA keys, SHA-256 signing, a 397-day server certificate validity, and a 3650-day CA validity. Nginx is configured for TLS 1.2 and TLS 1.3.

Before signing, the CA role validates each CSR's signature, Common Name, exact SAN list, critical `CA:FALSE` basic constraints, and `serverAuth` extended key usage. The TLS deployment secures the server private-key directory and file with root ownership and restrictive permissions.

See [TLS and PKI](tls-and-pki.md).

## Validation

The repository includes:

- Molecule verification for each of the eight roles.
- `tests/validation/security.yml` for selected checks on configured servers.
- GitHub Actions syntax checks and a matrix running all eight Molecule scenarios.

Run the server-side validation playbook with:

```bash
ansible-playbook tests/validation/security.yml
```

It checks Nginx and Fail2ban service state, UFW activation, selected effective SSH settings, certificate verification, HTTPS response, and HTTP redirection. It does not independently audit every firewall rule or every security control.

## Scope and Limitations

This is a baseline, not a guarantee of complete security. The project does not currently provide automated OS security upgrades, centralized logging, MFA, automated certificate renewal or revocation, vulnerability scanning, external penetration testing, or application-level security testing. Review the controls against the target environment's threat model before production use.
