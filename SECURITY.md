# Security Policy

## Supported Versions

| Version | Supported |
|---------|-----------|
| 1.x.x   | Yes       |
| < 1.0   | No        |

## Reporting a Vulnerability

We take the security of this Ansible role seriously. If you discover a security
vulnerability, please follow these steps.

### Do Not Open a Public Issue

Please do not report security vulnerabilities through public GitHub issues,
discussions, or pull requests.

### Report Privately

Email: **security@devopsgroup.sk**

Please include:

- Type of vulnerability (e.g. privilege escalation, information disclosure,
  credential leak, code injection)
- Full paths of source file(s) related to the vulnerability
- Location of the affected code (tag/branch/commit or direct URL)
- Step-by-step instructions to reproduce the issue
- Proof-of-concept or exploit code (if possible)
- Impact assessment, including how an attacker might exploit it

### Response Timeline

- Acknowledgement within **3 business days**
- Regular progress updates
- Security patch target: **30 days** for critical vulnerabilities
- Coordinated public disclosure (with your consent) once fixed

## Security Best Practices When Using This Role

### TLS

- TLS is enabled by default (`vault_tls_enabled: true`). Never disable it in
  production.
- Supply BYO certificates from a trusted CA. Role-generated self-signed certs
  (`vault_tls_generate_certs: true`) are for dev/test only.
- Ensure `vault_tls_leader_servername` matches a DNS SAN on every node cert
  (CN matching was removed in Go 1.17; mismatch causes an explicit x509 error).

### Key and token handling

- Init output (`vault_init_output_path`) is saved mode `0600` on the controller.
  Enable `vault_init_encrypt_output: true` (default) to encrypt it with
  `ansible-vault` immediately after writing.
- Never commit `.vault_init_*.json` files — they are listed in `.gitignore`.
- Use PGP key encryption (`vault_init_pgp_keys`) for air-gapped or multi-operator
  environments where `ansible-vault` is not sufficient.
- All tasks touching unseal keys, recovery keys, root token, or seal secrets run
  with `no_log: true`. Never add debug tasks that reference these values.

### Auto-unseal

- Recovery keys (returned by auto-unseal init) only gate root-token generation
  and rekey — they **cannot** unseal a cluster if the KMS is unavailable.
- KMS key deletion = permanent, irrecoverable cluster loss. Document this clearly
  and protect the KMS key accordingly.
- For Transit auto-unseal, use an orphan periodic token with **no max TTL** to
  prevent expiry-induced cluster lock-out.

### Swap

- `vault_manage_swap: true` (default) disables swap to prevent Vault key material
  being paged to disk when `vault_disable_mlock: true` (required for Raft). Do not
  disable swap management without an alternative mlock mechanism.

### Least privilege

- The `vault` service user and group are created with minimal permissions (no login
  shell, no home directory).
- `vault.env` (holds seal secrets as env vars) is mode `0640`, owner `vault:vault`.
- Private key is mode `0640`, owner `root:vault`.

## Security Updates

Updates will be announced via:

1. GitHub Security Advisories
2. Git tags with security notes
3. CHANGELOG.md with a `[SECURITY]` prefix

---

**Last Updated:** 2026-06-18
