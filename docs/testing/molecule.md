# Molecule Testing

## Overview

Molecule is used to test the project's Ansible roles independently in disposable container environments.

Each of the eight roles has a dedicated scenario under:

`roles/<role_name>/molecule/default/`

## Test Environment

| Component | Configuration |
|---|---|
| Driver | Podman |
| Container image | `geerlingguy/docker-ubuntu2404-ansible:latest` |
| Operating system | Ubuntu 24.04 |
| Verifier | Ansible |
| Init system | systemd |
| Execution mode | Rootless Podman |

The test containers are separate from the project's KVM-managed web servers.

## Scenario Structure

Each role's default scenario contains:

```text
roles/
└── common/
    └── molecule/
        └── default/
            ├── molecule.yml
            ├── converge.yml
            └── verify.yml
```

- `molecule.yml` defines the driver, test platform, and verifier
- `converge.yml` applies the role to the test instance
- `verify.yml` checks the resulting configuration

## Running Tests

Activate the project's Python virtual environment from the repository root:

```bash
source .venv/bin/activate
```

Move into the role you want to test:

```bash
cd roles/common
```

Run the complete Molecule lifecycle:

```bash
molecule test
```

Molecule creates the container, prepares it, applies the role, runs verification, and destroys the test instance.

To test another role, change to its directory before running the command.

## Useful Commands

Run the individual lifecycle stages when troubleshooting:

```bash
molecule create
molecule converge
molecule verify
molecule destroy
```

These commands operate on the current role's default scenario.

## Test Isolation

Molecule tests are designed to use disposable containers. They must not use the production inventory or target the actual KVM web servers.

Keep test configuration self-contained within each role's Molecule scenario.

## Troubleshooting

Check the following when a scenario fails:
- Podman is installed and available to the current user.
- The container image can be pulled.
- The container has the required systemd and cgroup support.
- The Python virtual environment is active.
- Required Ansible collections are installed.
- The role's converge and verify playbooks are valid.

## Current Scope

Molecule currently provides role-level testing. It does not replace full deployment testing on virtual machines or the project's security validation playbook.

A root-level scenario that tests the complete infrastructure deployment has not been implemented.
