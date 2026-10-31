# Integration Tests

This directory is reserved for automated tests that validate interactions between multiple Ansible roles and services.

## Current Status

A dedicated integration test suite has not yet been implemented.

Existing validation is provided by:

- Role-level Molecule scenarios under `roles/*/molecule/default/`.
- The security validation playbook at `tests/validation/security.yml`.
- The complete deployment playbook at `playbooks/site.yml`.

See [Integration Testing](../../docs/testing/integration-testing.md) for the planned test coverage and approach.

## Guidelines

Future integration tests should:

- Use a dedicated test inventory.
- Run against disposable infrastructure.
- Never target production KVM virtual machines.
- Avoid committing private keys, certificates, or credentials.
- Clean up test resources after execution.
