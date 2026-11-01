# Integration Testing

## Overview

Integration testing checks how multiple roles and services behave together. Relevant interactions include SSH access, UFW, Fail2ban, Nginx, and TLS.

## Current Status

The repository currently includes:

- Eight independent Molecule scenarios under `roles/*/molecule/default/`.
- GitHub Actions syntax checks and a Molecule matrix for all eight roles.
- A security-validation playbook at `tests/validation/security.yml`.
- A complete baseline deployment playbook at `playbooks/site.yml`.

A dedicated automated integration suite that provisions and tests the full stack on disposable virtual machines has not yet been implemented. The `tests/integration/` directory contains this guidance and a README, not an executable end-to-end suite.

## Existing Security Validation

After the web servers have been configured, run:

```bash
ansible-playbook tests/validation/security.yml
```

The playbook checks Nginx configuration and service state, Fail2ban and its SSH jail, UFW activation, selected effective SSH settings, certificate verification, HTTPS response, and HTTP-to-HTTPS redirection.

These checks validate selected aspects of an existing deployment. They do not provision the environment or test every firewall rule and security control.

## Planned Integration Coverage

| Scenario            | Expected behavior                                                                |
| ------------------- | -------------------------------------------------------------------------------- |
| Complete deployment | All baseline roles and TLS deployment complete successfully                      |
| Idempotence         | A second deployment produces no unintended changes                               |
| SSH hardening       | Automation access remains available and disallowed methods are rejected          |
| Firewall            | Required services are reachable and unsolicited inbound traffic is blocked       |
| Fail2ban            | SSH jail is active and uses the configured UFW action                            |
| Nginx               | Site is served and configuration is valid                                        |
| TLS                 | HTTPS certificate chains to the configured CA and matches the requested identity |
| HTTP redirect       | HTTP requests receive the configured redirect to HTTPS                           |
| Recovery behavior   | SSH/firewall changes do not leave the test environment inaccessible              |

## Recommended Test Environment

Use disposable Ubuntu 24.04 virtual machines or other dedicated test infrastructure. Do not use production KVM servers as disposable test targets.

Use a separate inventory, credentials, and CA for integration tests. Do not expose private keys, CA signing keys, or production inventory in CI logs or artifacts.

## Future Automation

A future integration workflow should:

1. Provision disposable test virtual machines.
2. Create a dedicated test inventory and credentials.
3. Apply `playbooks/site.yml`.
4. Run `tests/validation/security.yml`.
5. Check deployment idempotence.
6. Exercise SSH, firewall, HTTP, and HTTPS behavior.
7. Collect non-sensitive diagnostics.
8. Destroy test infrastructure even if a test fails.

The integration workflow should remain isolated from production systems and should never use production CA material.
