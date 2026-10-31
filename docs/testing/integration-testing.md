# Integration Testing

## Overview

Integration testing validates how multiple Ansible roles and infrastructure services work together.

Unlike isolated Molecule role tests, integration tests should exercise interactions between components such as SSH, UFW, Fail2ban, Nginx, and TLS.

## Current Status

The project currently has:

- Individual Molecule scenarios for all eight roles.
- A security validation playbook at `tests/validation/security.yml`.
- A complete deployment playbook at `playbooks/site.yml`.

A dedicated automated integration test suite has not yet been implemented.

## Planned Integration Test Coverage

The following scenarios are candidates for future integration tests:

| Scenario            | Expected behavior                                                                           |
| ------------------- | ------------------------------------------------------------------------------------------- |
| Complete deployment | All configured roles apply successfully                                                     |
| Idempotence         | A second deployment produces no unintended changes                                          |
| SSH hardening       | SSH accepts the configured automation account and rejects disallowed authentication methods |
| Firewall            | Required ports are accessible and other unsolicited inbound traffic is blocked              |
| Fail2ban            | SSH jail is active with the expected configuration                                          |
| Nginx               | Web server is active and serves the expected content                                        |
| TLS                 | HTTPS uses a valid certificate signed by the project's CA                                   |
| HTTP redirect       | HTTP requests are redirected to HTTPS                                                       |

## Test Environment

Integration tests should run against disposable Ubuntu 24.04 virtual machines or another dedicated test environment.

The existing KVM web servers must not be treated as disposable CI targets.

Use a separate test inventory and credentials. Do not expose private keys, CA private keys, or production inventory details in test logs or GitHub Actions artifacts.

## Manual Validation

The existing security validation playbook can be used to check selected deployed services and security settings:

```bash
ansible-playbook tests/validation/security.yml
```

The complete deployment can be applied using:

```bash
ansible-playbook playbooks/site.yml
```

These commands provide useful manual validation, but they do not constitute a separate automated integration test suite.

## Future Automation

A future integration workflow could:
1. Provision disposable test virtual machines.
2. Configure a dedicated Ansible inventory.
3. Apply playbooks/site.yml.
4. Run security validation.
5. Check deployment idempotence.
6. Collect logs and test results.
7. Destroy the test infrastructure.

Integration testing should remain isolated from production infrastructure and should clean up resources even when a test fails.
