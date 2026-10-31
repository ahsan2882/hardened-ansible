# Firewall Policy

## Overview

The `firewall` role manages the host-based firewall using Uncomplicated Firewall (UFW).

The policy is designed to expose only the services required by the managed web servers.

## Default Policy

| Traffic direction | Default policy |
|---|---|
| Incoming | Deny |
| Outgoing | Allow |
| Routed | Deny |

Unsolicited incoming traffic is denied unless an explicit rule allows it.

## Allowed Ports

The default configuration allows the following TCP ports:

| Port | Protocol | Purpose |
|---|---|---|
| 22 | TCP | SSH administration |
| 80 | TCP | HTTP access and HTTPS redirection |
| 443 | TCP | HTTPS access |

Additional ports can be configured through the firewall role variables when required.

## Configuration Management

The role:

1. Installs UFW.
2. Configures incoming, outgoing, and routed traffic policies.
3. Applies the configured allowed-port rules.
4. Configures firewall logging.
5. Enables UFW according to the role configuration.

The default logging level is `low`.

## Operational Considerations

- Ensure SSH is allowed before enabling the firewall.
- Confirm that the configured SSH port matches the OpenSSH configuration.
- Add required application ports explicitly rather than changing the default incoming policy.
- Consider outbound restrictions separately if the environment requires them.
- Review firewall rules before applying changes to remote production systems.

## Validation

The security validation playbook checks that UFW is active:

```bash
ansible-playbook tests/validation/security.yml
```

This check confirms firewall activation; it does not independently verify every configured rule or test network reachability.

## Scope and Limitations

The firewall role provides host-level traffic filtering. It does not configure:
- Cloud-provider network security groups.
- Network segmentation or subnet-level ACLs.
- Intrusion detection or deep packet inspection.
- Centralized firewall event collection.
- Application-specific network policies.

These controls should be considered separately when deploying the project in a larger infrastructure environment.
