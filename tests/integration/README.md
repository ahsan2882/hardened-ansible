# Integration Tests

This directory is reserved for automated tests that validate interactions between multiple Ansible roles and services.

## Current Status

A dedicated automated, VM-based integration suite has not yet been implemented.

The repository currently provides:

- Eight role-level Molecule scenarios under `roles/*/molecule/default/`.
- GitHub Actions syntax checks and a Molecule matrix for all eight roles.
- A live security-validation playbook at `tests/validation/security.yml`.
- The complete deployment playbook at `playbooks/site.yml`.

The security-validation playbook checks selected properties of an already-configured environment; it does not provision disposable infrastructure or constitute a full integration suite.

See [Integration Testing](../../docs/testing/integration-testing.md) for the planned coverage and approach.

## Guidelines for Future Tests

- Use a dedicated test inventory and credentials.
- Run against disposable infrastructure.
- Never target production KVM virtual machines.
- Do not reuse production CA keys or certificates.
- Keep private keys, certificates, credentials, and real inventory out of logs and artifacts.
- Test idempotence and interactions between SSH, UFW, Fail2ban, Nginx, and TLS.
- Clean up test resources even when a test fails.
