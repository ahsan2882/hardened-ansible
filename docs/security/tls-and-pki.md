# TLS and Public Key Infrastructure

## Overview

The project uses a locally managed Certificate Authority (CA) to issue certificates for Nginx web servers. Server private keys are generated and retained on their respective managed hosts; only CSRs are transferred to the control node for validation and signing.

## Components

- **Control node:** Stores the CA material, collects CSRs, validates requests, and signs certificates.
- **Managed web servers:** Generate their own server private keys and CSRs, then use deployed certificates with Nginx.
- **Local CA:** Issues server certificates after checking CSR identity and extensions.
- **Nginx:** Serves HTTPS and optionally redirects HTTP requests to HTTPS.

## Cryptographic Defaults

| Component                     | Default             |
| ----------------------------- | ------------------- |
| CA private key                | RSA 4096-bit        |
| Server private key            | RSA 3072-bit        |
| Digest                        | SHA-256             |
| CA certificate validity       | 3650 days           |
| Server certificate validity   | 397 days            |
| TLS protocols                 | TLS 1.2 and TLS 1.3 |
| Server CSR basic constraints  | Critical `CA:FALSE` |
| Server CSR extended key usage | `serverAuth`        |

## Certificate Workflow

The workflow is split into separate playbooks:

1. `ansible-playbook playbooks/generate-csrs.yml` generates server keys and CSRs on managed hosts and fetches the CSRs.
2. `ansible-playbook playbooks/local-ca.yml` creates the local CA.
3. `ansible-playbook playbooks/sign-csrs.yml` validates every configured CSR and signs them with the CA.
4. `ansible-playbook playbooks/deploy-tls.yml` deploys signed certificates and CA trust, then configures Nginx HTTPS and HTTP redirection.

The signing task validates the CSR signature, configured Common Name, exact SAN list, critical `CA:FALSE` constraint, and server-authentication EKU before the signing task is reached.

## Nginx TLS Configuration

The TLS deployment configures:

- The signed server certificate and server private-key paths.
- TLS 1.2 and TLS 1.3.
- Session caching and timeout.
- TLS session tickets disabled.
- The configured security response headers.
- Per-client request rate limiting, using the global zone created by the Nginx role.
- A managed HTTPS virtual host.
- An HTTP 301 redirect when `tls_http_redirect` is enabled.

Nginx configuration changes notify a handler that tests the configuration with `nginx -t` before reloading the service.

## Key and Certificate Permissions

The TLS role ensures the server private-key directory is owned by root with mode `0700` and the server private key is owned by root with mode `0600`.

The local CA workspace is created with mode `0700`; the CA private key is generated with mode `0600`. Keep the CA key and backups outside version control and protect them with access controls appropriate for a signing key.

## Trust Requirements

The CA certificate is installed into the managed server's system trust store. This does not automatically make other client machines trust the CA. Distribute the CA certificate through a trusted channel to clients that need to validate the server certificates.

The certificate SAN must match the hostname or IP address used by the client, and that name must resolve to the intended server.

## Validation

The server-validation playbook checks certificate verification against the configured CA certificate, an HTTPS response with certificate validation enabled, and HTTP redirection:

```bash
ansible-playbook tests/validation/security.yml
```

The TLS Molecule scenario also verifies the role's deployment behavior, including HTTPS configuration and redirect enable/disable behavior.

## Limitations

The project does not currently automate certificate renewal, certificate revocation, CRL distribution, OCSP, or CA rotation. Plan these operational processes before using the setup for long-lived production services.
