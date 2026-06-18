# ansible-role-hashicorp-vault

Production-ready Ansible role for installing and configuring **HashiCorp Vault** (OSS)
as a TLS-enabled, Raft integrated-storage HA cluster with idempotent init/unseal
(Shamir + auto-unseal), systemd hardening, audit devices, and Prometheus telemetry.

[![Ansible Galaxy](https://img.shields.io/badge/galaxy-devopsgroupeu.hashicorp__vault-blue)](https://galaxy.ansible.com/devopsgroupeu/hashicorp_vault)
[![License](https://img.shields.io/badge/license-Apache--2.0-blue)](LICENSE)

---

## Requirements

- **Ansible:** 2.19 or later
- **Collections:** `community.crypto >= 3.2.2`, `ansible.posix >= 1.5.0`
  (install via `ansible-galaxy collection install -r requirements.yml`)
- **Target OS:** Ubuntu 24.04 (noble), Debian 12/13 (bookworm/trixie), RHEL/Rocky/Alma 9
- **Vault version:** 2.0.3 (default; override via `vault_version`)

---

## Quick start

```yaml
# requirements.yml (consuming project)
collections:
  - name: community.crypto
    version: ">=3.2.2"
  - name: ansible.posix
    version: ">=1.5.0"
roles:
  - name: devopsgroupeu.hashicorp_vault
    version: "1.0.0"
```

### Single-node (dev/test — self-signed TLS, Shamir unseal)

```yaml
- name: Deploy Vault
  hosts: vault
  become: true
  roles:
    - role: devopsgroupeu.hashicorp_vault
      vars:
        vault_tls_generate_certs: true
        vault_init_key_shares: 1
        vault_init_key_threshold: 1
        vault_init_encrypt_output: false
```

### Three-node Raft cluster (production — BYO TLS)

```yaml
- name: Deploy Vault cluster
  hosts: vault           # [vault] group must contain all 3 hosts
  become: true
  roles:
    - role: devopsgroupeu.hashicorp_vault
      vars:
        # BYO certs: place them at vault_tls_{cert,key,ca}_file paths before running
        vault_tls_generate_certs: false
        vault_tls_cert_file: /opt/vault/tls/vault-cert.pem
        vault_tls_key_file: /opt/vault/tls/vault-key.pem
        vault_tls_ca_file: /opt/vault/tls/vault-ca.pem
        # Seal: Transit auto-unseal (deliver token via vault.env / no_log)
        vault_seal_type: transit
        vault_seal_transit_address: "https://unsealer.example.internal:8200"
        vault_seal_transit_key_name: autounseal
        vault_seal_transit_token: "{{ vault_transit_token }}"  # from Ansible Vault
```

---

## Role variables

Full variable reference with types, defaults, and descriptions is documented in
**`docs/CONFIGURATION.md`** (available from Phase 5 onward).

Key variables are declared in `defaults/main.yml` and validated in
`meta/argument_specs.yml` (1:1 mapping). Every public variable is prefixed `vault_`.

The most important defaults to be aware of:

| Variable | Default | Notes |
|---|---|---|
| `vault_version` | `2.0.3` | Pin your desired Vault version |
| `vault_install_method` | `package` | `package` or `binary` |
| `vault_tls_enabled` | `true` | TLS on by default — supply certs or set `vault_tls_generate_certs: true` |
| `vault_tls_generate_certs` | `false` | `true` for dev/test self-signed cert |
| `vault_seal_type` | `shamir` | Change to `transit`/`awskms`/`azurekeyvault`/`gcpckms` for auto-unseal |
| `vault_raft_peers` | `groups['vault']` | All nodes in the `[vault]` inventory group |
| `vault_init_bootstrap_node` | `vault_raft_peers[0]` | Where init is delegated |
| `vault_config_only` | `false` | Set `true` for template-only dry-run |

---

## Security notes

- **TLS is on by default.** Supply BYO certificates (`vault_tls_generate_certs: false`,
  the default) or enable role-generated self-signed certs for dev/test.
- **Init output** (`vault_init_output_path`) is saved mode `0600` on the controller
  and optionally encrypted with `ansible-vault encrypt` (`vault_init_encrypt_output: true`,
  the default). Never commit this file.
- **Unseal keys vs recovery keys:** under auto-unseal, Vault returns *recovery keys*,
  not unseal keys. Recovery keys only gate root-token generation and rekey — they cannot
  unseal a cluster if the KMS is unavailable. KMS key deletion = permanent cluster loss.
- **Transit unsealer token:** use an orphan periodic token with no max TTL to prevent
  expiry-induced cluster lock-out.
- All tasks touching unseal/recovery keys, root token, or seal secrets run with
  `no_log: true`.

---

## License

Apache-2.0 — see [LICENSE](LICENSE).

## Maintainer

[DevOpsGroup s.r.o.](https://devopsgroup.sk/) — info@devopsgroup.sk
