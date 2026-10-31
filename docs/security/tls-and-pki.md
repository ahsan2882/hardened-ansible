# TLS and Public Key Infrastructure

## Overview

Hardened Ansible uses a locally managed Certificate Authority (CA) to issue TLS certificates for its Nginx web servers.

The certificate workflow separates server private-key generation, CSR creation, CA signing, and certificate deployment.

## Architecture

The project uses the following components:

- **Control node:** Generates and stores the local CA and signs server certificate requests.
- **Managed web servers:** Generate their own private keys and Certificate Signing Requests (CSRs).
- **Local CA:** Signs CSRs and issues server certificates.
- **Nginx:** Uses the deployed server certificates to provide HTTPS.

The CA private key remains on the control node. Server private keys are generated and retained on their respective managed hosts.

## Cryptographic Defaults

| Component | Default |
|---|---|
| CA private key | RSA 4096-bit |
| Server private key | RSA 3072-bit |
| CSR digest | SHA-256 |
| Certificate validity | 397 days for issued server certificates |
| CA validity | 3650 days |
| TLS protocols | TLS 1.2 and TLS 1.3 |

The server certificate requests include the configured Common Name and Subject Alternative Names (SANs), including DNS names and IP addresses.

## Certificate Lifecycle

1. Generate a private key on each managed web server.
2. Create a CSR using the server's configured identity and SANs.
3. Fetch the CSR to the control node.
4. Create the local CA if it does not already exist.
5. Sign the collected CSRs using the CA.
6. Deploy the signed server certificates and CA certificate.
7. Update the managed server's trusted CA store.
8. Configure Nginx to serve HTTPS and redirect HTTP requests.

The CA certificate must also be trusted by clients that connect to the servers if certificate verification is expected to succeed.

## Project PKI Directory

The default CA workspace is located under the project directory:

```text
pki/
└── ca/
    ├── ca.key
    ├── ca.crt
    ├── requests/
    │   ├── web01.csr
    │   └── web02.csr
    └── signed/
        ├── web01.crt
        └── web02.crt
```

The exact files depend on the hosts configured for certificate issuance.

The `pki/` directory is excluded from Git. Do not commit the CA private key, server private keys, or other private credentials.

## Deployment and Validation

The TLS role deploys certificates and configures the Nginx HTTPS virtual host.

The security validation playbook checks:
- Nginx configuration validity and service status.
- Certificate verification against the configured CA certificate.
- Successful HTTPS response with certificate validation enabled.
- HTTP redirection to HTTPS.

Run validation with:

```bash
ansible-playbook tests/validation/security.yml
```

## Important Operational Considerations

- Back up the CA private key securely and separately from the repository.
- Restrict access to the CA private key and its workspace.
- Distribute the CA certificate to clients that need to trust the issued certificates.
- Protect server private keys with appropriate ownership and permissions.
- Plan certificate renewal and CA rotation before certificates expire.
- Review certificate identities and SANs before signing requests.

## Current Limitations

The current implementation does not provide:
- Automated certificate renewal and rotation.
- Certificate revocation or CRL distribution.
- OCSP responder support.
- Automated CA key backup.
- A separate approval process for signing CSRs.

The current `local_ca` role contains both CA creation and CSR signing tasks in its main task sequence. Consequently, the `local-ca.yml` and `sign-csrs.yml` playbooks currently execute the same complete role
task sequence rather than separate operations. This should be refactored into distinct task entry points.
