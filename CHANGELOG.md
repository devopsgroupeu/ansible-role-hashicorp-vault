# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- Full variable set in `defaults/main.yml` covering install, paths/users,
  listener/TLS, Raft, seal/auto-unseal, init/unseal, telemetry, audit,
  systemd hardening, and service control (§2 of design spec)
- `meta/argument_specs.yml` 1:1 with `defaults/main.yml`
- Cross-variable validation in `tasks/validate.yml` (seal-type required fields,
  peer list non-empty, bootstrap node ∈ peers, BYO TLS file presence)
- Galaxy-ready `meta/main.yml` (namespace `devopsgroupeu`, Apache-2.0,
  `min_ansible_version: "2.19"`, platforms: Ubuntu noble / Debian bookworm+trixie / EL 9)
- `requirements.yml` (community.crypto >=3.2.2, ansible.posix >=1.5.0)
- `requirements.txt` (molecule, ansible-lint, yamllint pinned)
- `handlers/main.yml`: Reload systemd daemon / Restart Vault / Reload Vault (SIGHUP)
- `.ansible-lint` (profile: production; enable_list: no-log-password, loop-var-prefix)
- `.yamllint`, `.editorconfig`, `.gitignore`, `.pre-commit-config.yaml`
- `README.md`, `LICENSE` (Apache-2.0), `CONTRIBUTING.md`, `SECURITY.md`

## Breaking changes vs prototype (`v1`)

This is a clean **breaking major** release. The prototype (`v1`) is a hardcoded
proof-of-concept; no migration path is provided.

Notable breaking changes:
- **TLS on by default** (`vault_tls_enabled: true`); prototype had `tls_disable = 1`.
- **All variable names changed** to `vault_*` prefix (no back-compat aliases).
  Prototype vars (`domain_name`, `vault_ips`, `vault_instances`, `dependencies`,
  `unseal_keys_dir_output`, `root_token_dir_output`) are removed.
- **No hardcoded leader** — `hashicorptest01` logic replaced by `retry_join`-all-peers
  with `vault_init_bootstrap_node = vault_raft_peers[0]`.
- **Secure key handling** — plaintext unseal keys on controller replaced by `no_log`
  + mode `0600` + optional `ansible-vault encrypt`.
- Namespace changed from `devopsgroup` to `devopsgroupeu`.
- `min_ansible_version` raised to `2.19`.
- FQCN for all modules; `loop:` replaces `with_*`; `include_tasks` for conditional
  includes; `notify:` replaces `when: x.changed`.

## [1.0.0] — target (planned)

First Galaxy-published stable release (all phases complete).

---

## Links

- [Ansible Galaxy](https://galaxy.ansible.com/devopsgroupeu/hashicorp_vault)
- [Repository](https://github.com/devopsgroupeu/ansible-role-hashicorp-vault)
- [Issues](https://github.com/devopsgroupeu/ansible-role-hashicorp-vault/issues)
