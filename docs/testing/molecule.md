# Molecule Testing

## Overview

Molecule tests each Ansible role independently in disposable containers. All eight project roles have a default scenario under:

`roles/<role_name>/molecule/default/`

## Test Environment

| Component        | Configuration                                  |
| ---------------- | ---------------------------------------------- |
| Driver           | Podman                                         |
| Image            | `geerlingguy/docker-ubuntu2404-ansible:latest` |
| Operating system | Ubuntu 24.04                                   |
| Verifier         | Ansible                                        |
| Init system      | systemd-capable container                      |
| Execution        | Rootless Podman locally                        |

The scenarios are isolated from the project's KVM-managed web servers.

## Scenario Structure

A typical scenario contains:

```text
roles/
└── nginx/
    └── molecule/
        └── default/
            ├── molecule.yml
            ├── converge.yml
            └── verify.yml
```

Some roles also include `prepare.yml` or `side_effect.yml` for setup and additional lifecycle checks.

- `molecule.yml` defines the driver, platform, provisioner, and verifier.
- `converge.yml` applies the role to the test instance.
- `verify.yml` checks the resulting configuration.
- `prepare.yml` prepares prerequisites where required.
- `side_effect.yml` exercises behavior that belongs after convergence/idempotence, where configured.

## Prerequisites

Create and activate the project virtual environment, then install Ansible, Molecule, the Podman plugin, and the collections:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install ansible-core==2.21.5 molecule==26.10.0 'molecule-plugins[podman]'
ansible-galaxy collection install -r requirements.yml -p ./collections
```

Ensure Podman is installed and usable by the current user.

## Run a Single Role

```bash
cd roles/nginx
molecule test
```

Molecule runs the scenario lifecycle, including instance creation, preparation (if defined), convergence, idempotence, verification, side effects (if defined), and cleanup.

## Run All Eight Roles

From the repository root:

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

The current role set is `common`, `fail2ban`, `firewall`, `local_ca`, `nginx`, `ssh_hardening`, `tls`, and `users`.

## Useful Lifecycle Commands

Run individual stages while troubleshooting:

```bash
molecule create
molecule converge
molecule idempotence
molecule verify
molecule destroy
```

Run these commands from the role directory whose scenario you want to inspect.

## Troubleshooting

Check that:

- Podman is installed and works for the current user.
- The required container image can be pulled.
- The container supports the systemd/cgroup configuration required by the scenario.
- The project virtual environment is active.
- Required Ansible collections are installed.
- The role's converge, verify, and optional lifecycle playbooks are valid.

## Scope

Molecule tests individual roles in isolation. It does not replace deployment testing on disposable virtual machines or the live security-validation playbook. A root-level Molecule scenario for the complete infrastructure stack is not currently present.
