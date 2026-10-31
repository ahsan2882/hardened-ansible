# Certificate Lifecycle

## Overview

Hardened Ansible uses a locally managed Certificate Authority (CA) to issue TLS certificates for its web servers.

The CA is managed on the Ansible control node, while each web server generates and retains its own TLS private key.

This design avoids transferring server private keys between machines.

## Certificate Components

| Component                 | Location                                             | Purpose                       |
| ------------------------- | ---------------------------------------------------- | ----------------------------- |
| CA private key            | Control node                                         | Signs server certificates     |
| CA certificate            | Control node and managed servers                     | Establishes certificate trust |
| Server private key        | Individual web server                                | Used by Nginx for TLS         |
| Server CSR                | Generated on web server, fetched to control node     | Requests a signed certificate |
| Signed server certificate | Control node before deployment; web server afterward | Identifies the web server     |

## Certificate Workflow

### 1. Generate Server Private Keys and CSRs

Run:

```bash
ansible-playbook playbooks/generate-csrs.yml
```

The `tls` role performs the following operations on each web server:
- Creates the TLS private-key directory.
- Generates an RSA private key.
- Generates a certificate signing request.
- Includes the configured Common Name (CN) and Subject Alternative Names (SANs).
- Fetches the CSR to the control node.

The server private key remains on the web server.

### 2. Create the Local Certificate Authority

Run:

```bash
ansible-playbook playbooks/local-ca.yml
```

The `local_ca` role:
- Creates the CA workspace.
- Generates an RSA 4096-bit private key.
- Generates a CA CSR.
- Creates a self-signed CA certificate.
- Configures a ten-year validity period.
- Sets CA basic constraints and certificate-signing key usage.

The CA private key is stored in the local PKI workspace.

### 3. Sign Server CSRs

Run:

```bash
ansible-playbook playbooks/sign-csrs.yml
```

The CA signs the collected CSRs using the `community.crypto.x509_certificate` module.

The resulting server certificates are written to the signed certificate workspace.

The configured certificate validity is 397 days.

### 4. Deploy Certificates:

Run:

```bash
ansible-playbook playbooks/deploy-tls.yml
```

The `tls` role's deployment tasks:
- Copy the signed server certificate to the configured certificate path.
- Install the local CA certificate into the system CA directory.
- Update the system CA trust store when the CA certificate changes.
- Deploy the HTTPS Nginx virtual host.
- Configure the HTTP-to-HTTPS redirect.

### 5. Configure Nginx

The HTTPS virtual host:
- Listens on TCP port 443.
- Uses the server's private key and signed certificate.
- Enables TLS 1.2 and TLS 1.3.
- Disables TLS session tickets.
- Configures session caching and timeout.
- Serves the managed website.

The HTTP virtual host redirects requests to HTTPS using HTTP 301 responses.

## PKI Workspace

The local CA defaults define the following workspace:

```text
pki/
└── ca/
    ├── ca.key
    ├── ca.crt
    ├── ca.csr
    ├── requests/
    │   ├── web01.csr
    │   └── web02.csr
    └── signed/
        ├── web01.crt
        └── web02.crt
```

This directory is generated at runtime and must not be committed to version control.

## Subject Alternative Names

The example inventory defines both DNS and IP SAN entries.

For example:

```yaml
tls_common_name: web01.hardened-ansible.test

tls_subject_alt_names:
  - DNS:web01.hardened-ansible.test
  - IP:192.0.2.10
```

Modern TLS clients validate hostnames against SAN entries. The SAN must match the hostname or IP address used by the client.

## Security Considerations

- Keep the CA private key on the control node.
- Restrict access to the CA private key and PKI workspace.
- Never commit private keys or generated PKI artifacts.
- Generate server private keys on the target servers.
- Verify certificates against the local CA.
- Back up the CA private key securely. Losing it prevents the existing CA from issuing replacement certificates.
- Protect CA backups separately from the Ansible repository.

## Operational Notes

The current implementation runs the complete `local_ca` role from both `local-ca.yml` and `sign-csrs.yml`. Since the role contains both CA creation and signing tasks, these playbooks are not yet isolated operations.

The certificate workflow should be refactored into separate CA initialization and CSR signing task entry points before relying on these playbooks as independent operational steps.
