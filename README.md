# Ansible Role - HashiCorp Vault

[![CI](https://github.com/devopsgroupeu/ansible-role-hashicorp-vault/actions/workflows/ci.yml/badge.svg)](https://github.com/devopsgroupeu/ansible-role-hashicorp-vault/actions/workflows/ci.yml)
[![Ansible Galaxy](https://img.shields.io/badge/Ansible%20Galaxy-devopsgroupeu.hashicorp--vault-blue?logo=ansible)](https://galaxy.ansible.com/ui/standalone/roles/devopsgroupeu/hashicorp-vault/)
[![License](https://img.shields.io/badge/license-Apache--2.0-blue)](LICENSE)
![GitHub Forks](https://img.shields.io/github/forks/devopsgroupeu/ansible-role-hashicorp-vault)
![GitHub Stars](https://img.shields.io/github/stars/devopsgroupeu/ansible-role-hashicorp-vault)
![GitHub Issues](https://img.shields.io/github/issues/devopsgroupeu/ansible-role-hashicorp-vault)

[![LinkedIn](https://img.shields.io/badge/linkedin-%230077B5.svg?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/company/devopsgroup8/)
![HashiCorp Vault](https://img.shields.io/badge/HashiCorp%20Vault-FFEC6E?logo=vault&logoColor=000&style=for-the-badge)

Production-ready Ansible role for installing and configuring **HashiCorp Vault** (OSS)
as a TLS-enabled, Raft integrated-storage HA cluster with idempotent init/unseal
(Shamir + auto-unseal), systemd hardening, audit devices, and Prometheus telemetry.

---

## Requirements

- **Ansible:** 2.19 or later (`ansible-core >= 2.19`)
- **Collections:** `community.crypto >= 3.2.2`
  Install via: `ansible-galaxy collection install -r requirements.yml`
- **Target OS:** Ubuntu 24.04 (noble), Debian 12/13 (bookworm/trixie), RHEL/Rocky/Alma 9
- **Vault version:** 2.0.3 (default; override via `vault_version`)
- **Docker (Molecule):** required for running tests locally
- **`become: true` at the play level** — the role does not set `become` per-task

---

## Quick start

### Install

```bash
ansible-galaxy collection install -r requirements.yml
ansible-galaxy role install devopsgroupeu.hashicorp-vault
```

Or pin a version from Git:

```yaml
# requirements.yml
roles:
  - name: devopsgroupeu.hashicorp-vault
    src: https://github.com/devopsgroupeu/ansible-role-hashicorp-vault.git
    version: "v1.0.0"
    scm: git
```

### Single-node (dev/test — self-signed TLS, Shamir unseal)

```yaml
- name: Deploy Vault
  hosts: vault
  become: true
  roles:
    - role: devopsgroupeu.hashicorp-vault
      vars:
        vault_tls_generate_certs: true
        vault_init_key_shares: 1
        vault_init_key_threshold: 1
        vault_init_encrypt_output: false
```

### Three-node Raft cluster (production — BYO TLS, Shamir)

```yaml
- name: Deploy Vault cluster
  hosts: vault           # [vault] group must contain all 3 hosts
  become: true
  roles:
    - role: devopsgroupeu.hashicorp-vault
      vars:
        vault_tls_generate_certs: false
        vault_tls_cert_file: /opt/vault/tls/vault-cert.pem
        vault_tls_key_file: /opt/vault/tls/vault-key.pem
        vault_tls_ca_file: /opt/vault/tls/vault-ca.pem
        vault_init_key_shares: 5
        vault_init_key_threshold: 3
        vault_init_encrypt_output: true
        vault_init_encrypt_password_file: "~/.vault-pass"
```

### Three-node Raft cluster with Transit auto-unseal

```yaml
- name: Deploy Vault cluster (auto-unseal)
  hosts: vault
  become: true
  roles:
    - role: devopsgroupeu.hashicorp-vault
      vars:
        vault_tls_generate_certs: false
        vault_seal_type: transit
        vault_seal_transit_address: "https://unsealer.example.internal:8200"
        vault_seal_transit_key_name: autounseal
        vault_seal_transit_token: "{{ vault_seal_transit_token }}"   # from Ansible Vault
        # With auto-unseal, init returns recovery keys (not Shamir unseal keys).
        # Recovery keys gate root-token generation and rekey only.
        vault_init_recovery_shares: 5
        vault_init_recovery_threshold: 3
        vault_init_encrypt_output: true
        vault_init_encrypt_password_file: "~/.vault-pass"
```

See `examples/autounseal_transit.yml` for the complete playbook including the
required unsealer Vault pre-requisites (transit enable, key creation, policy, and
orphan periodic token creation).

See `examples/` for all complete playbooks.

---

## Role variables

All 107 variables are documented in full in **[`docs/CONFIGURATION.md`](docs/CONFIGURATION.md)**.
Every variable is prefixed `vault_`, declared in `defaults/main.yml`, and validated
by `meta/argument_specs.yml` (1:1 mapping).

Key variables to be aware of:

| Variable | Default | Notes |
|---|---|---|
| `vault_version` | `"2.0.3"` | Pin your desired Vault version |
| `vault_install_method` | `"package"` | `package` or `binary` |
| `vault_tls_enabled` | `true` | **TLS on by default** — supply BYO certs or set `vault_tls_generate_certs: true` |
| `vault_tls_generate_certs` | `false` | `true` = self-signed cert (dev/test only) |
| `vault_tls_leader_servername` | `"vault.cluster.internal"` | Must be a DNS SAN on every node's cert |
| `vault_seal_type` | `"shamir"` | Change to `transit`/`awskms`/`azurekeyvault`/`gcpckms` for auto-unseal |
| `vault_raft_peers` | `groups['vault']` | All nodes in the `[vault]` inventory group |
| `vault_init_bootstrap_node` | `vault_raft_peers[0]` | Node where `vault operator init` runs |
| `vault_init_output_path` | `{{ playbook_dir }}/.vault_init_…json` | Mode `0600`; optionally encrypted |
| `vault_init_encrypt_output` | `true` | Encrypt init JSON with `ansible-vault` |
| `vault_disable_mlock` | `true` | Required for Raft/BoltDB mmap |
| `vault_manage_swap` | `true` | `swapoff -a` + purge fstab |
| `vault_config_only` | `false` | `true` = template-only dry-run (skip install/service/init) |

---

## Security notes

### TLS

TLS is **on by default**. Supply BYO certificates (`vault_tls_generate_certs: false`,
the default) or enable role-generated self-signed certs for dev/test only. The
`vault_tls_leader_servername` value **must be a DNS SAN** on every node's certificate
(CN matching was removed in Go 1.17); a mismatch causes an explicit x509 handshake
error during `retry_join`.

**Generated-cert mode (`vault_tls_generate_certs: true`) replicates the CA private
key to every node in the play.** This simplifies dev/test setups but means the CA
private key leaves the controller. Use BYO certificates (`vault_tls_generate_certs:
false`, the default) for all production deployments.

### Init output

The init file (`vault_init_output_path`) is saved mode `0600` on the Ansible
controller and optionally encrypted with `ansible-vault encrypt`
(`vault_init_encrypt_output: true`, the default). **Never commit this file.**
The filename includes `vault_cluster_id` to prevent multi-cluster overwrites.

### Unseal keys vs recovery keys

Under auto-unseal (`vault_seal_type != shamir`), Vault returns **recovery keys**,
not unseal keys. Recovery keys only gate root-token generation and rekey — they
**cannot unseal** a cluster if the KMS is unavailable. **KMS key deletion =
permanent, unrecoverable cluster loss.** Enable deletion protection on your KMS key.

### Transit unsealer token

Use an **orphan periodic token with no max TTL** for the transit unsealer. A
non-periodic token expiring while all Vault nodes are down causes permanent lock-out.
See `examples/autounseal_transit.yml` for the recommended creation command.

### Certificate validity (825 days)

The default `vault_tls_cert_validity: "+825d"` is a hard limit on **Apple platforms
only**. Linux systems and Vault itself accept longer periods. Adjust as required by
your PKI policy.

### Sensitive tasks

All tasks touching unseal/recovery keys, the root token, or seal credentials run
with `no_log: true`. A `-vvv` run will **not** expose these values.

### mlock and swap

`vault_disable_mlock: true` is required for Raft integrated storage (BoltDB uses
mmap). With mlock disabled, key material may be written to swap. The role runs
`swapoff -a` and purges fstab entries when `vault_manage_swap: true` (default).

### Handler note

The daemon-reload, restart, and reload handlers call `ansible.builtin.systemd`
which requires root. They rely on the consuming play declaring `become: true` at
the play level.

---

## Tests (Molecule)

```bash
# Install test dependencies
pip install -r requirements.txt

# Offline template validation (fast, no Vault install)
molecule test -s render

# Full single-node install + init + unseal + audit
molecule test -s default

# Three-node Raft HA cluster (heavy — run manually)
molecule test -s raft-ha
```

The CI pipeline (`ci.yml`) runs `render` and `default` on `debian12`, `ubuntu2404`,
and `rockylinux9` on every push/PR. The `raft-ha` scenario is manual
(workflow_dispatch) due to resource requirements.

---

## File layout

```
ansible-role-hashicorp-vault/
├── defaults/main.yml              # 107 public variables
├── vars/main.yml                  # OS-family maps (internal)
├── meta/main.yml                  # Galaxy metadata
├── meta/argument_specs.yml        # 1:1 with defaults (107 entries)
├── tasks/
│   ├── main.yml                   # orchestrator
│   ├── validate.yml               # cross-var assertions
│   ├── install.yml                # dispatch package/binary
│   ├── install_package.yml        # deb822_repository / yum_repository
│   ├── install_binary.yml         # GPG-verified zip
│   ├── config.yml                 # dirs, perms, vault.env, vault.hcl, vault.service
│   ├── tls.yml                    # generate (community.crypto) or BYO + trust store
│   ├── service.yml                # systemd start/enable
│   ├── init_unseal.yml            # idempotent init + unseal + raft verify
│   └── audit.yml                  # file audit device
├── handlers/main.yml              # daemon-reload / restart / reload
├── templates/
│   ├── vault.hcl.j2
│   ├── vault.env.j2
│   └── vault.service.j2
├── molecule/{default,render,raft-ha}/
├── examples/{single_node.yml,raft_cluster.yml,autounseal_transit.yml}
│   └── inventory/vault_cluster.yml
├── docs/{ARCHITECTURE.md,CONFIGURATION.md,INTEGRATION.md,TROUBLESHOOTING.md}
└── .github/workflows/{ci.yml,galaxy.yml}
```

---

## Contributing

We welcome contributions! Please see [CONTRIBUTING.md](CONTRIBUTING.md) to get started.

## Code of Conduct

Read our [Code of Conduct](CODE_OF_CONDUCT.md).

## Changelog

See the [GitHub Releases](https://github.com/devopsgroupeu/ansible-role-hashicorp-vault/releases) page for release history.

## License

```
Copyright 2025 DevOpsGroup

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```

---

> For more information or support, please refer to the official documentation or contact us at info@devopsgroup.sk
