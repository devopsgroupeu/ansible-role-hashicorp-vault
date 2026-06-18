# Configuration Reference

Full variable reference for the HashiCorp Vault role. All variables are defined
with their defaults in [`defaults/main.yml`](../defaults/main.yml) and validated
by [`meta/argument_specs.yml`](../meta/argument_specs.yml). Every public variable
is prefixed `vault_`.

---

## Table of Contents

- [Install](#install)
- [Paths and Users](#paths-and-users)
- [Listener / Addresses / UI](#listener--addresses--ui)
- [TLS](#tls)
- [Storage (Raft)](#storage-raft)
- [Seal / Auto-unseal](#seal--auto-unseal)
- [Init / Unseal](#init--unseal)
- [Telemetry / Logging / TTLs / Hardening](#telemetry--logging--ttls--hardening)
- [Audit](#audit)
- [systemd / Permissions](#systemd--permissions)
- [Service Control](#service-control)

---

## Install

| Variable | Default | Description |
|---|---|---|
| `vault_version` | `"2.0.3"` | Pinned Vault version for reproducible installs. Override to upgrade. |
| `vault_install_method` | `"package"` | `package` (HashiCorp APT/YUM repo, default) or `binary` (GPG-verified zip). |
| `vault_package_name` | `"vault"` | APT/YUM package name. Set to `vault-enterprise` for the ENT build. |
| `vault_apt_repo_url` | `"https://apt.releases.hashicorp.com"` | APT repository base URL. |
| `vault_apt_gpg_key_url` | `"https://apt.releases.hashicorp.com/gpg"` | APT repository GPG key URL. |
| `vault_rpm_repo_baseurl` | `"https://rpm.releases.hashicorp.com/RHEL/$releasever/$basearch/stable"` | RPM repository baseurl (contains `$releasever`/`$basearch`). |
| `vault_rpm_gpg_key_url` | `"https://rpm.releases.hashicorp.com/gpg"` | RPM repository GPG key URL. |
| `vault_binary_base_url` | `"https://releases.hashicorp.com/vault"` | Root URL for binary zip downloads (binary install method). |
| `vault_binary_gpg_key_url` | `"https://www.hashicorp.com/.well-known/pgp-key.txt"` | HashiCorp PGP key URL for verifying `SHA256SUMS.sig` (binary method). |
| `vault_binary_verify_gpg` | `true` | GPG-verify `SHA256SUMS.sig` before extracting. Set `false` only in air-gapped envs where you have already verified the checksum offline. |
| `vault_arch` | `"amd64"` / `"arm64"` | CPU architecture string for repo and binary paths. Auto-detected from `ansible_architecture`. |

---

## Paths and Users

| Variable | Default | Description |
|---|---|---|
| `vault_config_dir` | `"/etc/vault.d"` | Directory for `vault.hcl` and `vault.env`. |
| `vault_data_dir` | `"/opt/vault/data"` | Raft integrated storage data directory (role creates it). |
| `vault_tls_dir` | `"/opt/vault/tls"` | Directory for TLS material (cert, key, CA). |
| `vault_audit_dir` | `"/opt/vault/audit"` | Directory for the file audit device log. |
| `vault_user` | `"vault"` | Service user. Package install creates it; binary path creates it via tasks. |
| `vault_group` | `"vault"` | Service group. |
| `vault_bin_path` | `/usr/bin/vault` (package) / `/usr/local/bin/vault` (binary) | Vault binary path. Auto-computed from `vault_install_method`; override only when your distro places it elsewhere. |

---

## Listener / Addresses / UI

| Variable | Default | Description |
|---|---|---|
| `vault_listen_address` | `"0.0.0.0:8200"` | API listener bind address and port. |
| `vault_cluster_listen_address` | `"0.0.0.0:8201"` | Raft cluster listener bind address and port (Vault-internal mTLS). |
| `vault_api_addr` | `"https://{{ ansible_default_ipv4.address }}:8200"` | Advertised API address. Must use `https://` scheme; override per-node via `host_vars` in multi-node deployments. |
| `vault_cluster_addr` | `"https://{{ ansible_default_ipv4.address }}:8201"` | Advertised cluster address for Raft replication. Must use `https://` scheme. |
| `vault_ui` | `true` | Enable the Vault web UI. |
| `vault_cluster_name` | `""` | Human-readable cluster label in `vault.hcl`. Empty = omit (Vault auto-generates). |

---

## TLS

TLS is **on by default** (`vault_tls_enabled: true`). Two modes:

- **BYO (production default, `vault_tls_generate_certs: false`):** supply cert/key/CA at the configured paths before running the role.
- **Generated (dev/test, `vault_tls_generate_certs: true`):** role builds a CA + per-node cert via `community.crypto` ownca.

> **SAN requirement:** `vault_tls_leader_servername` must be a **DNS SAN** on every
> node's certificate (CN matching was removed in Go 1.17). Mismatch = explicit x509
> handshake error on `retry_join`. The SAN must also include `serverAuth` **and**
> `clientAuth` extended key usage (Raft mTLS uses the API cert for cluster joins).

| Variable | Default | Description |
|---|---|---|
| `vault_tls_enabled` | `true` | Enable TLS on the API listener. **Never disable in production.** |
| `vault_tls_cert_file` | `"{{ vault_tls_dir }}/vault-cert.pem"` | Server certificate file path (full chain PEM). |
| `vault_tls_key_file` | `"{{ vault_tls_dir }}/vault-key.pem"` | Server private key file path. |
| `vault_tls_ca_file` | `"{{ vault_tls_dir }}/vault-ca.pem"` | CA certificate file path (used by clients and `retry_join`). |
| `vault_tls_min_version` | `"tls12"` | Minimum TLS version. Choices: `tls12`, `tls13`. |
| `vault_tls_max_version` | `"tls13"` | Maximum TLS version. Choices: `tls12`, `tls13`. |
| `vault_tls_cipher_suites` | GCM+ChaCha list (no 3DES) | TLS 1.2 cipher suites. **Emitted only when `vault_tls_min_version == vault_tls_max_version == tls12`**; ignored under TLS 1.3. |
| `vault_tls_disable_client_certs` | `true` | Disable mutual TLS (client certificate) verification for API clients. Keep `true` unless you specifically require mTLS for API consumers. |
| `vault_tls_generate_certs` | `false` | When `true`, role generates a self-signed CA + per-node cert via `community.crypto`. **Dev/test only.** |
| `vault_tls_cert_key_type` | `"ECC"` | Key type for role-generated certificates. Choices: `ECC` (P-384, default), `RSA`. |
| `vault_tls_cert_curve` | `"secp384r1"` | ECC curve for generated certificates (when `vault_tls_cert_key_type=ECC`). |
| `vault_tls_cert_size` | `4096` | RSA key size for generated certificates (when `vault_tls_cert_key_type=RSA`). |
| `vault_tls_cert_validity` | `"+825d"` | Certificate validity period (`ownca not_after` format). Note: 825 days is a hard limit only on Apple platforms; other platforms accept longer periods. |
| `vault_tls_leader_servername` | `"vault.cluster.internal"` | Shared DNS SAN that must appear on every node's cert and must match `leader_tls_servername` in `retry_join` blocks. Does not need to resolve in DNS. |
| `vault_tls_extra_sans` | `[]` | Additional `DNS:` or `IP:` SANs for role-generated certificates. Example: `["DNS:vault.example.com", "IP:10.0.0.1"]`. |
| `vault_tls_install_ca_to_trust_store` | `true` | Install the role-generated CA into the OS trust store (APT: `/usr/local/share/ca-certificates/`; RHEL: `/etc/pki/ca-trust/source/anchors/`). |

---

## Storage (Raft)

Only Raft integrated storage is supported by this role.

| Variable | Default | Description |
|---|---|---|
| `vault_storage` | `"raft"` | Storage backend. Only `raft` is supported. |
| `vault_raft_node_id` | `"{{ inventory_hostname }}"` | Deterministic Raft node ID. **Never use a random value** — random IDs cause split-brain on restart. |
| `vault_raft_peers` | `groups['vault'] \| default([inventory_hostname])` | List of inventory hostnames forming the cluster. Used to render `retry_join` blocks and to gate init/unseal delegation. |
| `vault_raft_performance_multiplier` | `1` | Raft performance multiplier. `1` = on-prem LAN tight timing (recommended). Use `5` for cloud/WAN deployments with higher latency. |
| `vault_raft_extra_parameters` | `{}` | Extra `key = value` pairs injected verbatim into the `storage "raft" {}` stanza (e.g. autopilot tuning). Dict of `{key: value}`. |

---

## Seal / Auto-unseal

Default is Shamir (no `seal` stanza in `vault.hcl`). Set `vault_seal_type` to
configure auto-unseal. **Sensitive credentials (tokens, keys) are delivered via
`vault.env` EnvironmentFile — never written into `vault.hcl`.**

> **Recovery keys vs unseal keys:** with auto-unseal, `vault operator init` returns
> *recovery keys*, not unseal keys. Recovery keys only gate root-token generation and
> rekey — they cannot unseal the cluster if the KMS is unavailable. KMS key deletion
> or permanent KMS unavailability = **permanent cluster loss**.

### Common

| Variable | Default | Description |
|---|---|---|
| `vault_seal_type` | `"shamir"` | Seal type. Choices: `shamir`, `transit`, `awskms`, `azurekeyvault`, `gcpckms`. |

### Transit

| Variable | Default | Description |
|---|---|---|
| `vault_seal_transit_address` | `""` | Unsealer Vault address (DNS name / VIP). **Never use a node IP for HA.** |
| `vault_seal_transit_key_name` | `"autounseal"` | Transit encryption key name on the unsealer Vault. |
| `vault_seal_transit_mount_path` | `"transit/"` | Transit secrets engine mount path on the unsealer Vault. |
| `vault_seal_transit_namespace` | `""` | ENT namespace on the unsealer Vault. Empty = omit from config. |
| `vault_seal_transit_tls_ca_cert` | `""` | CA certificate path for TLS verification of the unsealer Vault. |
| `vault_seal_transit_disable_renewal` | `false` | Disable automatic token renewal. Keep `false` (default) to avoid token expiry. |
| `vault_seal_transit_token` | `""` | Token for the transit unsealer. Delivered via `vault.env` (`no_log`); never written into `vault.hcl`. **Use an orphan periodic token with no max TTL.** |

### AWS KMS

| Variable | Default | Description |
|---|---|---|
| `vault_seal_awskms_region` | `"us-east-1"` | AWS KMS region. |
| `vault_seal_awskms_kms_key_id` | `""` | AWS KMS key ID, ARN, or alias. |
| `vault_seal_awskms_access_key` | `""` | AWS access key. Prefer IAM instance role; inject via environment instead of setting this. |
| `vault_seal_awskms_secret_key` | `""` | AWS secret key. Prefer IAM instance role. |

### Azure Key Vault

| Variable | Default | Description |
|---|---|---|
| `vault_seal_azurekeyvault_tenant_id` | `""` | Azure Active Directory tenant ID. |
| `vault_seal_azurekeyvault_vault_name` | `""` | Azure Key Vault name. |
| `vault_seal_azurekeyvault_key_name` | `""` | Azure Key Vault key name. |
| `vault_seal_azurekeyvault_client_id` | `""` | Azure service principal client ID. Omit when using Managed Service Identity (MSI). |
| `vault_seal_azurekeyvault_client_secret` | `""` | Azure service principal client secret. Omit when using MSI. |

### GCP Cloud KMS

| Variable | Default | Description |
|---|---|---|
| `vault_seal_gcpckms_project` | `""` | GCP project ID for Cloud KMS. |
| `vault_seal_gcpckms_region` | `""` | Key-ring **location** (e.g. `global`). This is **not** a compute region. |
| `vault_seal_gcpckms_key_ring` | `""` | GCP Cloud KMS key ring name. |
| `vault_seal_gcpckms_crypto_key` | `""` | GCP Cloud KMS crypto key name. |
| `vault_seal_gcpckms_credentials` | `""` | Path to a GCP service account JSON key file. Omit on GCE instances with an attached service account. |

---

## Init / Unseal

The role performs idempotent initialization and unseal on the bootstrap node
(`vault_raft_peers[0]`). All tasks touching unseal/recovery keys, the root token,
or seal secrets run with `no_log: true`.

| Variable | Default | Description |
|---|---|---|
| `vault_init` | `true` | Run idempotent Vault initialization on the bootstrap node. Skipped when Vault is already initialized. |
| `vault_unseal` | `true` | Run idempotent Shamir unseal after init. No-op under auto-unseal (KMS unseals on startup). |
| `vault_init_key_shares` | `5` | Number of Shamir key shares to generate. |
| `vault_init_key_threshold` | `3` | Minimum shares required to reconstruct the master key. |
| `vault_init_recovery_shares` | `5` | Recovery key shares (auto-unseal init). |
| `vault_init_recovery_threshold` | `3` | Recovery key threshold (auto-unseal init). |
| `vault_init_bootstrap_node` | `"{{ vault_raft_peers[0] }}"` | Inventory hostname of the bootstrap node where init is delegated. **Never hardcode a hostname here.** |
| `vault_init_pgp_keys` | `[]` | Base64-encoded *binary* PGP public keys (`gpg --export <fp> \| base64`) for encrypting individual unseal/recovery shares. Empty = plaintext shares (protect with `vault_init_encrypt_output`). |
| `vault_init_output_path` | `"{{ playbook_dir }}/.vault_init_{{ vault_cluster_id }}.json"` | Controller-side path where the init JSON is saved (mode `0600`). Includes `vault_cluster_id` to prevent multi-cluster overwrites. |
| `vault_init_encrypt_output` | `true` | Encrypt the saved init JSON with `ansible-vault encrypt` after writing. Strongly recommended. |
| `vault_init_encrypt_password_file` | `""` | Path to a password file on the Ansible controller for `ansible-vault encrypt --vault-password-file`. Encryption is skipped (with a warning) when empty, to avoid hanging unattended runs. |
| `vault_cluster_id` | `"{{ vault_raft_peers[0] }}"` | Stable identifier used in the init output filename. Override to a human-readable name for multi-cluster playbooks. |

---

## Telemetry / Logging / TTLs / Hardening

| Variable | Default | Description |
|---|---|---|
| `vault_telemetry_enabled` | `true` | Render a `telemetry {}` stanza in `vault.hcl` for Prometheus scraping via `/v1/sys/metrics`. |
| `vault_telemetry_prometheus_retention_time` | `"24h"` | Prometheus metric retention time in Vault's internal ring buffer. |
| `vault_telemetry_disable_hostname` | `true` | Omit the hostname label from metrics for cleaner dashboards. |
| `vault_telemetry_unauthenticated_metrics_access` | `false` | Allow unauthenticated `/v1/sys/metrics` access. Rendered **inside `listener "tcp" > telemetry {}`** (not top-level). Keep `false` to require a scrape token. |
| `vault_log_level` | `"info"` | Vault log verbosity. Choices: `trace`, `debug`, `info`, `warn`, `error`. |
| `vault_log_format` | `"json"` | Log format. `json` produces structured logs suitable for Loki / Elasticsearch. |
| `vault_default_lease_ttl` | `"1h"` | Default lease TTL. Overrides Vault's built-in 768h default. |
| `vault_max_lease_ttl` | `"24h"` | Maximum lease TTL. Caps lease renewal duration. |
| `vault_disable_mlock` | `true` | Required `true` for Raft integrated storage (BoltDB uses mmap). No deprecation exists. Pair with `vault_manage_swap: true`. |
| `vault_manage_swap` | `true` | Run `swapoff -a` and purge `/etc/fstab` swap entries to compensate for `disable_mlock=true`. |
| `vault_enable_file_permissions_check` | `true` | Set `VAULT_ENABLE_FILE_PERMISSIONS_CHECK=true` in `vault.env` so Vault validates file permissions on startup. |
| `vault_extra_config` | `{}` | Free-form extra top-level `vault.hcl` key=value pairs. Dict of `{key: value}`. |

---

## Audit

The role enables a file audit device after Vault is initialized. Requires the root
token from the init output (`vault_init_output_path`).

| Variable | Default | Description |
|---|---|---|
| `vault_audit_enabled` | `true` | Enable a file audit device after Vault is initialized. Gated on `vault_init: true`. |
| `vault_audit_path` | `"file/"` | Vault audit device mount path (used with `vault audit enable -path=…`). |
| `vault_audit_file_path` | `"{{ vault_audit_dir }}/vault_audit.log"` | Audit log file path on the target host. |
| `vault_audit_options` | `{}` | Extra options passed to `vault audit enable` as `key=value` pairs. |

---

## systemd / Permissions

| Variable | Default | Description |
|---|---|---|
| `vault_systemd_limit_nofile` | `65536` | `LimitNOFILE` for the Vault systemd unit. |
| `vault_systemd_restart_sec` | `5` | `RestartSec` (seconds). |
| `vault_systemd_timeout_stop_sec` | `30` | `TimeoutStopSec` (seconds). |
| `vault_config_dir_mode` | `"0750"` | Mode for the config directory (`/etc/vault.d`). Owner `vault:vault`. |
| `vault_config_file_mode` | `"0640"` | Mode for config files (`vault.hcl`, `vault.env`). Owner `vault:vault`. |
| `vault_data_dir_mode` | `"0750"` | Mode for the Raft data directory (`/opt/vault/data`). Owner `vault:vault`. |
| `vault_tls_key_mode` | `"0640"` | Mode for the TLS private key file. Owner `root:vault`. |
| `vault_tls_cert_mode` | `"0644"` | Mode for TLS cert and CA files. Owner `root:root`. |
| `vault_manage_selinux` | `true` | Run `restorecon -R` on RHEL-family after creating or modifying files. No-op on Debian/Ubuntu. |

---

## Service Control

| Variable | Default | Description |
|---|---|---|
| `vault_service_enabled` | `true` | Enable `vault.service` at boot. |
| `vault_service_state` | `"started"` | Desired systemd service state after configuration. |
| `vault_config_only` | `false` | Render-only mode: skip install, service management, and init/unseal. Used by the `render` Molecule scenario for offline template validation. |

---

## Notes

- **`become: true` required at the play level.** The role does not set `become`
  on individual tasks; the consuming play must declare `become: true`.
- **BYO TLS file existence is checked at runtime.** If `vault_tls_generate_certs: false`
  (the default) and the cert/key/CA files do not exist at the configured paths, the
  role fails early in the `validate` stage.
- **Handlers require play-level `become: true`.** The daemon-reload and restart
  handlers call `ansible.builtin.systemd` which requires root. They will silently
  fail if `become` is not set at the play level.
