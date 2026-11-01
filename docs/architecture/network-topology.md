# Network Topology

## Overview

The project uses a centralized Ansible control node to configure one or more Ubuntu web servers. The control node connects to managed servers over SSH. Clients connect to the web servers over HTTP and HTTPS.

The example inventory uses documentation-only IP addresses. Replace them in the local inventory.

| Host              | Inventory group | Example address      |
| ----------------- | --------------- | -------------------- |
| `ansible-control` | Control node    | Environment-specific |
| `web01`           | `webservers`    | `192.0.2.10`         |
| `web02`           | `webservers`    | `192.0.2.11`         |

## Logical Topology

```text
                 Ansible Control Node
             (Ansible, inventory, local CA)
                         |
                    SSH / TCP 22
                         |
             +-----------+-----------+
             |                       |
             v                       v
       +-----------+           +-----------+
       |   web01   |           |   web02   |
       |-----------|           |-----------|
       | OpenSSH   |           | OpenSSH   |
       | UFW       |           | UFW       |
       | Fail2ban  |           | Fail2ban  |
       | Nginx     |           | Nginx     |
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

## Host Firewall Policy

The default firewall policy is:

| Direction | Default |
| --------- | ------- |
| Incoming  | Deny    |
| Outgoing  | Allow   |
| Routed    | Deny    |

The default allowed inbound TCP services are:

| Port                             | Service | Policy         |
| -------------------------------- | ------- | -------------- |
| Configured SSH port (default 22) | SSH     | UFW rate limit |
| 80                               | HTTP    | Allow          |
| 443                              | HTTPS   | Allow          |

The SSH port is shared through `system_ssh_port` and used by the SSH-hardening, firewall, and Fail2ban configuration. HTTP and HTTPS ports are configurable in the relevant role defaults.

## Management Traffic

The control node connects to managed servers using SSH public-key authentication and the configured `ansible` account. The SSH role restricts permitted users and disables password authentication and root login by default.

Keep a working administrative session and recovery access available when changing SSH or firewall settings.

## Client Traffic

- HTTP requests are redirected to HTTPS when `tls_http_redirect` is enabled.
- HTTPS requests are served by Nginx using a certificate signed by the local CA.
- Nginx applies configured response headers and per-client request rate limiting.
- Clients must trust the project's CA certificate for normal certificate validation.

The example server names are `web01.hardened-ansible.test` and `web02.hardened-ansible.test`. Configure DNS or local name resolution to map these names to the corresponding server addresses.

## Certificate and DNS Requirements

The certificate SAN must contain the hostname or IP address used by the client. The requested name must resolve to the intended server, and the client must trust the issuing CA certificate.

The project installs the CA certificate into the managed servers' system trust stores. Other client machines need their own CA trust configuration.

## Scope

This diagram describes host-level access and service flows. It does not provision cloud security groups, subnet ACLs, routing, DNS, load balancers, or network segmentation.
