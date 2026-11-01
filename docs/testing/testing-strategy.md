# Testing Strategy

## Overview

The project uses several complementary validation layers. They cover role behavior in disposable containers, syntax validation in CI, and selected checks against already-configured managed servers.

## Test Layers

| Layer               | Purpose                                                       | Location / execution                               |
| ------------------- | ------------------------------------------------------------- | -------------------------------------------------- |
| Role testing        | Test each role independently in a disposable Ubuntu container | `roles/*/molecule/default/`                        |
| Playbook syntax     | Parse and validate project playbooks                          | GitHub Actions workflow                            |
| Security validation | Check selected controls on configured servers                 | `tests/validation/security.yml`                    |
| Integration testing | Exercise interactions across roles on disposable VMs          | Planned; not yet implemented as an automated suite |

## Molecule Role Testing

Each of the eight roles has a default Molecule scenario. Scenarios use Podman, a systemd-capable Ubuntu 24.04 image, and Ansible verification.

Run a role's complete lifecycle from its directory:

```bash
source .venv/bin/activate
cd roles/common
molecule test
```

To run all scenarios from the repository root:

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

## GitHub Actions CI

`.github/workflows/ci.yml` runs on pushes and pull requests targeting `main`.

It contains two jobs:

1. **Playbook syntax checks:** installs Ansible Core and required collections, then runs `ansible-playbook --syntax-check` against the main deployment, certificate workflow, and security-validation playbooks using the example inventory.
2. **Molecule matrix:** runs `molecule test` independently for `common`, `fail2ban`, `firewall`, `local_ca`, `nginx`, `ssh_hardening`, `tls`, and `users`.

The workflow currently uses Python 3.12, Ansible Core 2.21.5, Molecule 26.10.0, and Podman.

CI tests syntax and isolated role behavior. It does not provision or deploy to the project's real KVM web servers.

## Security Validation on Managed Hosts

The playbook `tests/validation/security.yml` checks selected live configuration and service behavior:

- Nginx configuration and service status.
- Fail2ban service and SSH jail status.
- UFW activation.
- Selected effective SSH settings from `sshd -T`.
- Server certificate verification against the configured CA.
- HTTPS response with certificate validation enabled.
- HTTP redirect response.

Run it from the repository root after the target servers have been configured:

```bash
ansible-playbook tests/validation/security.yml
```

This is a validation playbook, not a substitute for a complete integration test suite or external security assessment.

## Integration Testing Status

The `tests/integration/` directory currently contains guidance only. An automated multi-role integration suite that provisions disposable VMs, deploys the full stack, verifies cross-service behavior, and destroys the VMs has not yet been implemented.

Future integration automation should use a dedicated inventory and credentials, never target production infrastructure, test idempotence and service interactions, and clean up resources even after failures.

## Isolation and Safety

- Molecule scenarios must use disposable containers.
- CI must use the committed example inventory, not a real environment inventory.
- Live validation requires an explicitly selected test or managed environment.
- Never expose private keys, CA signing material, credentials, or production inventory in test logs or artifacts.
- Review SSH and firewall changes carefully to avoid locking out administrators.

## Current Limitations

- Molecule verifies roles independently; it does not prove that a full VM deployment works end to end.
- The live validation playbook checks selected controls rather than every security setting.
- No automated VM-based integration suite is currently present.
- External vulnerability scanning and penetration testing are outside the current workflow.
