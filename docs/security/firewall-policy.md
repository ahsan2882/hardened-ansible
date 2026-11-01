# Firewall Policy

## Overview

The `firewall` role manages host-level firewall rules using UFW. The default policy denies unsolicited inbound traffic and allows only the services required for SSH management and web access.

## Default Policy

| Direction | Policy |
| --------- | ------ |
| Incoming  | Deny   |
| Outgoing  | Allow  |
| Routed    | Deny   |

## Default Inbound Rules

| Port                              | Protocol | Service | Rule    |
| --------------------------------- | -------- | ------- | ------- |
| `system_ssh_port` (22 by default) | TCP      | SSH     | `limit` |
| 80                                | TCP      | HTTP    | `allow` |
| 443                               | TCP      | HTTPS   | `allow` |

The SSH port is centralized in `system_ssh_port` and is also used by the SSH-hardening and Fail2ban roles. Keep these settings aligned if changing the SSH port.

## Configuration

The role:

1. Installs UFW.
2. Sets incoming, outgoing, and routed default policies.
3. Applies the configured inbound port rules.
4. Configures firewall logging (default: `low`).
5. Enables UFW when `firewall_enabled` is true.

Allowed ports and policies can be customized through the role defaults or inventory variables.

## Apply the Firewall Role

The firewall can be applied independently:

```bash
ansible-playbook playbooks/firewall.yml
```

It is also included in `playbooks/site.yml`.

## Operational Considerations

- Confirm SSH access and the correct SSH port are allowed before enabling UFW.
- Keep an existing administrative session open while changing remote firewall rules.
- Verify access from a second session before closing the existing one.
- Add required application ports explicitly.
- Consider outbound restrictions separately where the environment requires them.
- UFW does not replace cloud security groups, network ACLs, or network segmentation.

## Validation

The security-validation playbook checks that UFW is active:

```bash
ansible-playbook tests/validation/security.yml
```

The firewall Molecule scenario verifies role behavior in its isolated test container. The live validation playbook's UFW check confirms activation; it does not test every rule or network reachability from an external client.
