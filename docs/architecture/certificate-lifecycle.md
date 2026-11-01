# Certificate Lifecycle

## Overview

The project uses a local Certificate Authority (CA) to issue TLS certificates for managed Nginx servers. Each managed server generates and retains its own private key. The control node receives only the CSR, validates it, and signs it with the CA.

The lifecycle is separated into four playbooks so that key/CSR generation, CA initialization, signing, and deployment are distinct operations.

## Components

| Component                 | Location                                   | Purpose                                  |
| ------------------------- | ------------------------------------------ | ---------------------------------------- |
| CA private key            | Ansible control node                       | Signs server certificates                |
| CA certificate            | Control node and deployed trust stores     | Establishes trust in issued certificates |
| Server private key        | Corresponding managed server               | Used by Nginx                            |
| Server CSR                | Created on server, fetched to control node | Requests a signed certificate            |
| Signed server certificate | Created on control node, then deployed     | Identifies the server                    |

## 1. Generate Server Keys and CSRs

```bash
ansible-playbook playbooks/generate-csrs.yml
```

The `tls` role:

- Creates the private-key and CSR directories.
- Generates an RSA 3072-bit server private key with restrictive permissions.
- Creates a CSR using the configured Common Name and Subject Alternative Names.
- Adds critical `CA:FALSE` basic constraints and the `serverAuth` extended key usage.
- Fetches the CSR to the control node.

The private key remains on the managed server. The default CSR collection directory is under `pki/ca/requests/`.

## 2. Create the Local CA

```bash
ansible-playbook playbooks/local-ca.yml
```

The `local_ca` role's `create` task file:

- Creates the CA workspace with restrictive directory permissions.
- Generates an RSA 4096-bit CA private key.
- Creates a CA CSR with critical `CA:TRUE` basic constraints and certificate-signing key usage.
- Creates a self-signed CA certificate.

The default CA validity is 3650 days. The CA private key is sensitive and must not be committed, placed in an ordinary project archive, or exposed in logs. Store protected backups separately.

## 3. Validate and Sign CSRs

```bash
ansible-playbook playbooks/sign-csrs.yml
```

The `local_ca` role's `sign` task file inspects every configured server CSR before signing any of them. It checks:

- The CSR signature is valid.
- The CSR Common Name matches the host's configured `tls_common_name`.
- The CSR SAN list exactly matches the configured `tls_subject_alt_names`.
- Basic constraints are present, critical, and specify `CA:FALSE` rather than `CA:TRUE`.
- Extended key usage includes TLS server authentication.

If a CSR fails validation, the assertion task fails before the signing task runs. Validated CSRs are signed using the local CA. The default server certificate validity is 397 days.

## 4. Deploy Certificates and Configure Nginx

```bash
ansible-playbook playbooks/deploy-tls.yml
```

The `tls` deployment tasks:

- Ensure the server private-key directory is owned by root with mode `0700`.
- Ensure the existing server private key is owned by root with mode `0600`.
- Copy the signed server certificate to the configured certificate path.
- Install the CA certificate into `/usr/local/share/ca-certificates/`.
- Run `update-ca-certificates` when the deployed CA certificate changes.
- Deploy and enable the HTTPS virtual host.
- Deploy and enable the HTTP-to-HTTPS redirect when `tls_http_redirect` is enabled.
- Remove the redirect site and its symlink when redirection is disabled.

Nginx configuration changes notify a handler that runs `nginx -t` before reloading Nginx.

The HTTPS virtual host defaults to TLS 1.2 and TLS 1.3, disables TLS session tickets, configures session caching, applies security headers and rate limiting, and denies access to hidden files.

## PKI Workspace

The default CA workspace is under `pki/ca/`:

```text
pki/
└── ca/
    ├── ca.key
    ├── ca.csr
    ├── ca.crt
    ├── requests/
    │   ├── web01.csr
    │   └── web02.csr
    └── signed/
        ├── web01.crt
        └── web02.crt
```

The filenames under `requests/` and `signed/` are based on inventory hostnames. The directory is runtime data and must remain excluded from version control and untrusted archives.

## Subject Alternative Names

The example inventory configures both DNS and IP SANs:

```yaml
tls_common_name: web01.hardened-ansible.test
tls_subject_alt_names:
  - DNS:web01.hardened-ansible.test
  - IP:192.0.2.10
```

Clients validate the requested hostname or IP against the certificate SAN. Ensure the inventory identity, DNS resolution, and address used by clients agree.

## Operational Considerations and Limitations

- Protect the CA private key and its backups with strict access controls.
- Distribute the CA certificate to client devices that need to validate the issued server certificates. Installing it on the servers does not automatically make unrelated clients trust it.
- Protect server private keys on their respective hosts.
- Review each CSR identity and SAN before signing.
- The project does not currently automate certificate renewal, revocation, CRL distribution, or OCSP.
- CA rotation and certificate replacement require an operational procedure; they are not automated.
