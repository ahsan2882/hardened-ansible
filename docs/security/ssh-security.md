# SSH Security Configuration

## Overview

The `ssh_hardening` role configures OpenSSH on managed web servers using a dedicated configuration file:

`/etc/ssh/sshd_config.d/99-ansible-hardening.conf`

The role validates the configuration before applying it and reloads SSH when changes are detected.

## Authentication Policy

The default configuration applies the following restrictions:

| Setting | Value | Purpose |
|---|---|---|
| PermitRootLogin | no | Prevents direct root SSH login |
| PasswordAuthentication | no | Disables password-based SSH authentication |
| PubkeyAuthentication | yes | Requires public-key authentication |
| KbdInteractiveAuthentication | no | Disables keyboard-interactive authentication |
| MaxAuthTries | 3 | Limits authentication attempts per connection |
| AllowUsers | ansible | Restricts SSH access to the automation account |
| Port | 22 | Uses the default SSH port |

These values are configurable through role defaults and inventory variables.

## Configuration Management

The role:

1. Installs the OpenSSH server package.
2. Ensures `/run/sshd` exists.
3. Deploys the managed SSH hardening configuration.
4. Validates the resulting configuration using `sshd -t`.
5. Enables and starts the SSH service.
6. Reloads SSH when the managed configuration changes.

The configuration is deployed with restrictive file permissions.

## Operational Considerations

- Ensure public-key authentication works before disabling password authentication.
- Confirm the automation account is included in `AllowUsers`.
- Keep an existing administrative session open while applying SSH changes.
- Verify connectivity from a separate session before closing the existing session.
- Confirm the firewall permits the configured SSH port.

## Validation

The project's security validation playbook checks the effective SSH configuration using `sshd -T`.

It verifies:

- Root login is disabled.
- Password authentication is disabled.
- Public-key authentication is enabled.
- Keyboard-interactive authentication is disabled.
- Maximum authentication attempts are limited to three.
- SSH access is restricted to the `ansible` user.

Run the validation playbook with:

```bash
ansible-playbook tests/validation/security.yml
```

## Scope and Limitations

This configuration provides a baseline for SSH access hardening. It does not implement:
- SSH certificate-based authentication.
- Multi-factor authentication.
- Centralized SSH audit logging.
- Automatic SSH key rotation.
- Restrictions based on source IP addresses.

These controls can be introduced according to the environment's security requirements.
