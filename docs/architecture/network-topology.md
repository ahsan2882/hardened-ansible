# Network Topology

## Overview

Hardened Ansible uses a centralized Ansible control node to configure multiple remote web servers.

The control node communicates with managed servers over SSH. Web servers expose HTTP and HTTPS services to clients.

The example deployment contains:

| Host | Inventory group | Example IP |
|---|---|---|
| ansible-control | Control node | Environment-specific |
| web01 | webservers | 192.0.2.10 |
| web02 | webservers | 192.0.2.11 |

The addresses above are documentation-only examples. Replace them with the actual addresses in the local inventory.

## Logical Topology

```text
                  Ansible Control Node
                 (Automation and Local CA)
                          |
                          |
                    SSH / TCP 22
                          |
              +-----------+-----------+
              |                       |
              v                       v
        +-----------+           +-----------+
        |   web01   |           |   web02   |
        |           |           |           |
        | SSH       |           | SSH       |
        | Nginx     |           | Nginx     |
        | Fail2ban  |           | Fail2ban  |
        | UFW       |           | UFW       |
        +-----------+           +-----------+
              ^                       ^
              |                       |
          HTTP/HTTPS              HTTP/HTTPS
          TCP 80/443              TCP 80/443
              |                       |
              +-----------+-----------+
                          |
                       Clients
```

## Network Access Policy
The firewall role configures UFW with the following default policies:

| Traffic direction | Default policy |
| ----------------- | -------------- |
| Incoming          | Deny           |
| Outgoing          | Allow          |
| Routed            | Deny           |

The following inbound TCP ports are allowed:

| Port | Service | Purpose                                      |
| ---- | ------- | -------------------------------------------- |
| 22   | SSH     | Ansible management and administrative access |
| 80   | HTTP    | Redirect clients to HTTPS                    |
| 443  | HTTPS   | Secure web access                            |

All other inbound traffic is denied by default.

## Management Traffic

The Ansible control node connects to the managed servers using SSH public-key authentication.

The configured automation account is `ansible`.

The SSH hardening role disables password authentication and root login, enables public-key authentication, and restricts SSH access to the configured allowed users.

## Client Traffic

Clients access the web servers using their configured DNS names.
- HTTP requests on TCP port 80 receive a permanent redirect to HTTPS.
- HTTPS requests use TLS certificates signed by the project's local CA.
- Nginx serves the managed landing page over HTTPS.

The example DNS names are:
- `web01.hardened-ansible.test`
- `web02.hardened-ansible.test`

These names must resolve to the corresponding server addresses in the client environment.

## DNS and Certificate Requirements

The example inventory defines DNS names and IP addresses in each server certificate's Subject Alternative Name (SAN) extension.

For successful HTTPS validation:
1. The requested hostname must resolve to the intended server.
2. The certificate must contain the requested hostname or IP address in its SAN extension.
3. The client must trust the local CA certificate.

The project deploys the CA certificate to managed web servers. Other clients that access the web servers must also trust that CA certificate independently.
