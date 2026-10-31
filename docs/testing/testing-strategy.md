# Testing Strategy

## Overview

Hardened Ansible uses multiple testing levels to validate role behavior, infrastructure integration, and security configuration.

The testing strategy aims to catch configuration errors before changes are applied to the managed web servers.

## Testing Levels

| Level                 | Purpose                                                        | Location                                      |
| --------------------- | -------------------------------------------------------------- | --------------------------------------------- |
| Role testing          | Tests individual roles in isolated containers                  | `roles/*/molecule/default/`                   |
| Integration testing   | Tests interactions between roles and infrastructure components | `tests/integration/`                          |
| Security validation   | Checks important security settings on managed hosts            | `tests/validation/security.yml`               |
| Deployment validation | Checks the complete deployment and service behavior            | `playbooks/site.yml` and validation playbooks |


## Role Testing with Molecule

Each of the eight Ansible roles has its own Molecule scenario.

The scenarios use:

- Podman as the container driver.
- Ubuntu 24.04 as the test environment.
- Ansible as the verifier.
- A systemd-capable container configuration.

A role's Molecule scenario follows the standard lifecycle:

1. Create the test instance.
2. Prepare the environment.
3. Apply the role through the converge playbook.
4. Verify the resulting configuration.
5. Destroy the test instance.

Run a role's complete test lifecycle from its directory:

```bash
cd roles/common
molecule test
```

Replace `common` with another role name to test that role.

## Security Validation

The security validation playbook checks the deployed web servers.

It currently verifies:
- Nginx configuration validity and service status.
- Fail2ban service status and SSH jail configuration.
- UFW activation.
- Effective SSH authentication settings.
- TLS certificate verification against the configured CA.
- HTTPS response status.
- HTTP-to-HTTPS redirection.

Run it from the project root:

```bash
ansible-playbook tests/validation/security.yml
```

## Integration Testing

Integration tests are intended to verify behavior across multiple roles and services.

The `tests/integration/` directory is reserved for these tests. A separate automated integration test suite has not yet been implemented.

## Test Environment Isolation

Molecule tests use disposable Podman containers rather than the actual KVM web server VMs.

This keeps role-level testing separate from the running infrastructure.

Do not point test playbooks or automated CI jobs at production inventory. Use a dedicated test inventory and disposable targets.

### Current Limitations

- Role-level Molecule tests do not replace testing on real virtual machines.
- The integration test suite is not yet implemented.
- Security validation checks selected controls rather than every security setting.
- CI automation is not yet configured.
- External vulnerability scanning and penetration testing are outside the current test suite.
