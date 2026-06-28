# Troubleshooting

Common issues and resolutions for the `devopsgroupeu.hashicorp-vault` role.

---

## Table of Contents

- [Sealed cluster recovery](#sealed-cluster-recovery)
- [Auto-unseal: recovery keys cannot unseal — KMS down](#auto-unseal-recovery-keys-cannot-unseal--kms-down)
- [Transit auto-unseal: token expired or revoked](#transit-auto-unseal-token-expired-or-revoked)
- [TLS / x509 / SAN errors](#tls--x509--san-errors)
- [mlock and swap](#mlock-and-swap)
- [SIGHUP reload race](#sighup-reload-race)
- [Raft join failures](#raft-join-failures)
- [Package vs binary path issues](#package-vs-binary-path-issues)
- [Init output written unencrypted / encryption skipped](#init-output-written-unencrypted--encryption-skipped)
- [Useful commands](#useful-commands)

---

## Sealed cluster recovery

**Symptom:** `vault status` returns `sealed: true` after a node restart; the cluster
has a leader but individual nodes are sealed.

**Cause:** Shamir unseal keys must be applied on every restart. The role's unseal
tasks guard against re-unsealing an already-unsealed node (idempotent status check),
but they only run during the Ansible play. After an OS reboot the node starts sealed.

**Resolution:**

```bash
# Retrieve unseal key from the init file on the controller
ansible-vault view .vault_init_<cluster_id>.json --vault-password-file ~/.vault-pass

# On the sealed node (run for each required share until threshold is met)
export VAULT_ADDR=https://127.0.0.1:8200
export VAULT_CACERT=/opt/vault/tls/vault-ca.pem
vault operator unseal <unseal_key_1>
# ... repeat for each threshold share

# Verify
vault status
```

For production, consider Transit auto-unseal to eliminate manual unseal on restart
(see `examples/autounseal_transit.yml`).

---

## Auto-unseal: recovery keys cannot unseal — KMS down

**This is a critical operational constraint. Read carefully.**

With auto-unseal (`vault_seal_type: transit|awskms|azurekeyvault|gcpckms`), Vault
`operator init` returns **recovery keys**, not unseal keys.

**Recovery keys can:**
- Gate generation of a new root token (`vault operator generate-root`)
- Gate rekey operations (`vault operator rekey`)

**Recovery keys cannot:**
- Unseal the cluster if the KMS (unsealer) is unavailable
- Reconstruct the encryption key if the KMS key is deleted

**KMS key deletion = permanent, unrecoverable cluster loss.** There is no fallback.

**Mitigations:**
- Protect the KMS key with deletion protection / soft-delete where available.
- For AWS KMS: enable key deletion waiting period (7–30 days).
- For Azure Key Vault: enable soft-delete and purge protection.
- For GCP Cloud KMS: key-version destruction is scheduled (default 24-hour delay
  before permanent deletion). Increase the scheduled duration to at least 30 days.
  Consider enforcing the `constraints/cloudkms.disableBeforeDestroy` organisation
  policy to prevent accidental key destruction. Monitor all KMS operations with
  Cloud Audit Logs to receive early warning of key state changes.
  **Recovery keys cannot reconstruct the encryption key if the Cloud KMS key is
  destroyed — this is permanent, unrecoverable cluster loss.**
- For Transit (Vault-to-Vault): keep the unsealer Vault cluster healthy; back up its
  storage.
- Test your KMS recovery procedure before it is needed in production.

**If the KMS is temporarily unavailable** and all Vault nodes restart, the cluster
is sealed and cannot be unsealed until the KMS is restored. Vault nodes are
**read-only** while sealed — no writes or new leases.

---

## Transit auto-unseal: token expired or revoked

**Symptom:** After a node restart (or all nodes restarting), Vault does not
auto-unseal. The service starts but remains sealed with errors such as
`error performing token check` or `transit seal: error decrypting wrapping key`
in the logs.

**Diagnosis:**

```bash
# On the affected node, inspect the Vault service logs for transit seal errors
journalctl -u vault -n 100 --no-pager | grep -iE 'transit|seal|token|auth'
```

Look for messages indicating that the token used for the transit seal is invalid,
expired, has been revoked, or has reached its max TTL.

**Cause:** The transit unsealer token stored in `vault.env` (the `VAULT_TOKEN`
passed to the transit seal stanza) has expired or been revoked on the unsealer
Vault cluster. Vault cannot decrypt its wrapping key without a valid token.

**Remediation:**

1. On the unsealer Vault cluster, create a new orphan periodic token:

   ```bash
   vault token create \
     -orphan \
     -policy=autounseal \
     -period=24h \
     -display-name=vault-autounseal
   ```

2. Update the token in the secrets source used to populate `vault.env` on the
   sealed cluster (e.g. re-encrypt with Ansible Vault and update the group variable,
   or update the external secret). The token must land in `vault.env` as the value
   supplied to `vault_seal_transit_token`.

3. Re-run the role (or manually edit `{{ vault_config_dir }}/vault.env` on each
   node), then restart the Vault service:

   ```bash
   systemctl restart vault
   vault status
   ```

**Prevention:** Use an orphan periodic token with `period=24h` and **no max TTL**.
A non-periodic token or a token with `max_ttl` set will eventually expire even if
auto-renewed. Set `vault_seal_transit_disable_renewal: false` (the default) to keep
Vault auto-renewing the token while the service runs.

---

## TLS / x509 / SAN errors

### `x509: certificate is valid for X, not Y`

Vault's `leader_tls_servername` (set via `vault_tls_leader_servername`) must match
a **DNS SAN** on the server certificate. CN matching was removed in Go 1.17.

**Check SANs on the certificate:**
```bash
openssl x509 -in /opt/vault/tls/vault-cert.pem -noout -ext subjectAltName
```

**Fix:** add the missing hostname to `vault_tls_extra_sans` (generated mode) or
regenerate your BYO cert with the correct SANs. For `retry_join` you need:
- `DNS:{{ vault_tls_leader_servername }}` (e.g. `vault.cluster.internal`)
- `DNS:{{ inventory_hostname }}` for each node
- `IP:{{ ansible_default_ipv4.address }}` for each node
- `IP:127.0.0.1`

### `x509: certificate specifies an incompatible key usage`

The certificate must include both `serverAuth` and `clientAuth` in `extendedKeyUsage`.
Raft mTLS uses the API certificate for cluster join. Role-generated certs set both
by default. BYO certs must be regenerated if `clientAuth` is missing.

### `tls: failed to verify certificate: x509: certificate signed by unknown authority`

The CA on `vault_tls_ca_file` is not trusted. For `retry_join`, all nodes must share
the same CA. Check that `vault_tls_ca_file` is the correct CA and that
`vault_tls_install_ca_to_trust_store: true` was applied.

---

## mlock and swap

**Symptom:** Vault fails to start with:
```
Error initializing core: Failed to lock memory: operation not permitted
```

**Cause:** `disable_mlock = false` (the Vault default) requires `CAP_IPC_LOCK`.
This role sets `vault_disable_mlock: true` by default (required for Raft/BoltDB
mmap), and the systemd unit grants `AmbientCapabilities=CAP_IPC_LOCK` anyway.

**If you set `vault_disable_mlock: false`** on a system where the service user lacks
`CAP_IPC_LOCK`, Vault will fail to start. The capability is granted via
`AmbientCapabilities` in the templated unit, so this should not normally occur.

**Swap is a security concern with `disable_mlock: true`:** with mlock disabled, key
material may be written to swap. The role sets `vault_manage_swap: true` by default
to run `swapoff -a` and purge fstab entries. If you manage swap separately, set
`vault_manage_swap: false`.

---

## SIGHUP reload race

**Symptom:** `vault.service` crashes or enters a failed state shortly after a
certificate rotation (reload via SIGHUP).

**Cause:** Vault's SIGHUP reload (`ExecReload=/bin/kill --signal HUP $MAINPID`)
reloads TLS cert/key *content at the configured paths*, audit FDs, log level, and
license. It does **not** reload listener addresses, cert paths, or general
configuration. There is a known race (HashiCorp issue #27100) where reloading
immediately after start can cause instability.

**Fix:**
- Never trigger the reload handler immediately after service start. The role
  implements a readiness wait before any reload.
- For config changes (not just cert rotation): use restart, not reload.
- For cert rotation: overwrite the cert and key files at the same paths, then
  reload (`systemctl reload vault`).

---

## Raft join failures

**Symptom:** Only one or two nodes appear in `vault operator raft list-peers`.

**Cause:** Common causes:
1. `retry_join` cannot resolve the peer hostname or IP.
2. TLS SAN mismatch on `leader_tls_servername`.
3. Nodes started sequentially with long gaps — `retry_join` retries internally but
   may time out if the bootstrap node is not yet ready.

**Diagnosis:**
```bash
journalctl -u vault -n 200 --no-pager | grep -i raft
```

**Checks:**
- Confirm all nodes in `vault_raft_peers` are reachable on port 8200 from each other.
- Confirm `vault_tls_leader_servername` is a DNS SAN on every node's certificate.
- Confirm `vault_api_addr` is correctly set per node (not the same value on all nodes).

---

## Package vs binary path issues

### Package install: `vault` command not found

The HashiCorp package installs `vault` to `/usr/bin/vault`. If you have overridden
`vault_bin_path` incorrectly, tasks will fail with `vault: command not found`.

Check:
```bash
which vault
rpm -ql vault | grep bin  # RHEL
dpkg -L vault | grep bin  # Debian/Ubuntu
```

### Binary install: binary wiped on upgrade

The binary install method places `vault` at `/usr/local/bin/vault`. Linux package
managers do not manage this path; upgrades require re-running the role with the new
`vault_version`. The role GPG-verifies the new binary before installing it.

**Note:** `setcap` (used to grant `CAP_IPC_LOCK` on some older setups) is wiped when
the binary is replaced. This role uses `AmbientCapabilities` in the systemd unit
instead of `setcap` to survive binary replacements.

---

## Init fails: encryption requested but no password file set

**Symptom:** The play fails on the bootstrap node, before the init file is
written, with:

```
vault_init_encrypt_output is true but vault_init_encrypt_password_file is
not set. Refusing to write the init file (root token + unseal/recovery keys)
in plaintext to <path> on the Ansible controller. Set
vault_init_encrypt_password_file to a controller-side ansible-vault password
file, or explicitly set vault_init_encrypt_output: false to accept plaintext
storage.
```

**Cause:** `vault_init_encrypt_output: true` (the default) but
`vault_init_encrypt_password_file` is empty (also the default). The role
**fails closed**: rather than silently writing the root token and unseal/recovery
keys as plaintext JSON, it aborts *before* any init material is written to the
controller.

**Fix:** provide the password file path to enable encryption:
```yaml
vault_init_encrypt_password_file: "~/.vault-pass"
```

Or, for ephemeral/test runs where you accept plaintext storage:
```yaml
vault_init_encrypt_output: false
```

Regardless of the encryption setting, **never commit the init file** — it contains
unseal keys (Shamir) or recovery keys plus the initial root token.

---

## Useful commands

```bash
# Service status
systemctl status vault
journalctl -u vault -n 100 --no-pager

# Vault status (unsealed and initialized)
export VAULT_ADDR=https://127.0.0.1:8200
export VAULT_CACERT=/opt/vault/tls/vault-ca.pem
vault status

# Raft peers (requires root or operator token)
vault operator raft list-peers -format=json | jq .

# Autopilot health
vault operator raft autopilot state -format=json | jq .

# Active audit devices
vault audit list -format=json | jq .

# TLS certificate SANs
openssl x509 -in /opt/vault/tls/vault-cert.pem -noout -ext subjectAltName

# Check vault.hcl syntax (Vault parses it at start)
vault server -config=/etc/vault.d/vault.hcl -dev 2>&1 | head -20

# Reload TLS certs in place (no downtime)
systemctl reload vault
```
