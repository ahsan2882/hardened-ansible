# SSH Security Configuration

## Overview

The `ssh_hardening` role configures OpenSSH through:

`/etc/ssh/sshd_config.d/00-ansible-hardening.conf`

The early filename is intentional: it helps the managed settings take precedence over later drop-ins that may define the same options. The role validates the template with `sshd -t` before installing it and reloads SSH when relevant changes occur.

## Default Authentication and Access Policy

| Setting                        | Default                | Purpose                                      |
| ------------------------------ | ---------------------- | -------------------------------------------- |
| `PermitRootLogin`              | `no`                   | Disables direct root SSH login               |
| `PasswordAuthentication`       | `no`                   | Disables password authentication             |
| `PubkeyAuthentication`         | `yes`                  | Enables public-key authentication            |
| `KbdInteractiveAuthentication` | `no`                   | Disables keyboard-interactive authentication |
| `MaxAuthTries`                 | `3`                    | Limits authentication attempts               |
| `AllowUsers`                   | `ansible`              | Restricts permitted SSH users                |
| `Port`                         | `system_ssh_port` (22) | Uses the shared configured SSH port          |

## Session and Forwarding Defaults

The role also defaults to:

| Setting                 | Default       |
| ----------------------- | ------------- |
| `X11Forwarding`         | `no`          |
| `AllowAgentForwarding`  | `no`          |
| `AllowTcpForwarding`    | `no`          |
| `PermitTunnel`          | `no`          |
| `PermitEmptyPasswords`  | `no`          |
| `LoginGraceTime`        | `30` seconds  |
| `ClientAliveInterval`   | `300` seconds |
| `ClientAliveCountMax`   | `2`           |
| `MaxSessions`           | `5`           |
| `PermitUserEnvironment` | `no`          |

These values are configurable through role defaults and inventory variables.

## Configuration Management

The role:

1. Installs OpenSSH server.
2. Ensures `/run/sshd` exists.
3. Deploys the managed drop-in with restrictive permissions.
4. Validates the configuration using `sshd -t`.
5. Removes the obsolete `99-ansible-hardening.conf` project drop-in.
6. Removes a conflicting `AllowUsers` directive from `50-cloud-init.conf` when that file exists.
7. Ensures the SSH service is enabled and running and reloads it when configuration changes.

## Safe Deployment

SSH hardening can lock out remote administrators if the automation key or allowed-user list is incorrect.

- Configure and verify the automation account's public key before applying the role.
- Confirm the account is included in `AllowUsers`.
- Confirm the firewall permits the configured SSH port.
- Keep an existing administrative session open while applying changes.
- Verify a new SSH connection from a separate session before closing the original one.
- Ensure console or other recovery access is available.

## Validation

The security-validation playbook uses `sshd -T` to check selected effective settings:

```bash
ansible-playbook tests/validation/security.yml
```

It checks root login, password authentication, public-key authentication, maximum authentication attempts, and the configured allowed user. The SSH Molecule scenario provides additional role-level verification.

## Scope and Limitations

The current role does not configure SSH certificate authentication, MFA, centralized SSH audit logging, automated key rotation, or source-address restrictions. Add these controls separately if required by the environment.
