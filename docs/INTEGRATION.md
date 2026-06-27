# Integration Guide

How to consume the `devopsgroupeu.hashicorp-vault` role in other projects and how
it fits into the OpenPrime on-premise stack alongside the RKE2 and HAProxy roles.

---

## Table of Contents

- [Consuming the role via requirements.yml](#consuming-the-role-via-requirementsyml)
- [OpenPrime on-premise context](#openprime-on-premise-context)
- [Relationship to sibling roles](#relationship-to-sibling-roles)
- [Vault as a secrets backend for Kubernetes](#vault-as-a-secrets-backend-for-kubernetes)
- [External Secrets Operator (ESO)](#external-secrets-operator-eso)
- [Vault Agent](#vault-agent)
- [Post-role workflow](#post-role-workflow)

---

## Consuming the role via requirements.yml

### From Ansible Galaxy (after first publish)

```yaml
# requirements.yml in the consuming project
collections:
  - name: community.crypto
    version: ">=3.2.2"

roles:
  - name: devopsgroupeu.hashicorp-vault
    version: "1.0.0"
```

Install:

```bash
ansible-galaxy collection install -r requirements.yml
ansible-galaxy role install -r requirements.yml
```

### From Git source (pre-publish / pinned commit)

```yaml
# requirements.yml — git source
roles:
  - name: devopsgroupeu.hashicorp-vault
    src: https://github.com/devopsgroupeu/ansible-role-hashicorp-vault.git
    version: "feat/galaxy-readiness-v2"   # or a git tag: v1.0.0
    scm: git
```

Install:

```bash
ansible-galaxy role install -r requirements.yml -p roles/
```

The consuming playbook must set `roles_path` to include the install directory,
or use `ANSIBLE_ROLES_PATH` in the environment.

---

## OpenPrime on-premise context

The `hashicorp_vault` role is the **secrets backend** for the OpenPrime on-premise
platform. It is deployed before the application layer and before the Kubernetes
cluster's secrets operators are configured. The typical provisioning order is:

1. **Network / load-balancing tier** — `devopsgroupeu.haproxy-keepalived` provides
   the VIP that Vault clients resolve. The Vault cluster sits behind the VIP.
2. **Vault cluster** — `devopsgroupeu.hashicorp-vault` installs, configures, and
   initialises Vault on 3 nodes. Init output (root token + unseal/recovery keys) is
   saved to the Ansible controller, encrypted with `ansible-vault`.
3. **Kubernetes cluster** — `devopsgroupeu.rke2` provisions the RKE2 cluster.
   Vault is registered as a secrets backend during or after cluster bootstrapping.
4. **Secrets operators** — External Secrets Operator (ESO) or Vault Agent Injector
   are deployed into the cluster; they are configured to authenticate against Vault
   using Kubernetes service accounts.

---

## Relationship to sibling roles

### `devopsgroupeu.haproxy-keepalived`

HAProxy fronts the Vault API (port 8200) with a VIP. The Vault role's
`vault_api_addr` should point to the HAProxy VIP (not an individual node IP):

```yaml
# group_vars/vault/main.yml
vault_api_addr: "https://vault.example.internal:8200"
vault_tls_leader_servername: "vault.example.internal"
```

The TLS certificate must carry `vault.example.internal` as a DNS SAN. HAProxy
forwards TCP (mode tcp) without TLS termination so that Vault handles its own TLS.

### `devopsgroupeu.rke2`

The RKE2 role provisions the Kubernetes cluster. Vault integration happens
post-cluster via either the External Secrets Operator or the Vault Agent Injector.
The Vault role does not modify the Kubernetes cluster directly.

---

## Vault as a secrets backend for Kubernetes

After the Vault role completes, configure the Kubernetes authentication method:

```bash
# Enable the Kubernetes auth method
vault auth enable kubernetes

# Configure it with the RKE2 cluster endpoint
vault write auth/kubernetes/config \
  kubernetes_host="https://$(kubectl config view --raw -o json | jq -r '.clusters[0].cluster.server')" \
  kubernetes_ca_cert=@/var/lib/rancher/rke2/server/tls/server-ca.crt

# Create a policy for the secrets operator
vault policy write eso-read - <<'EOF'
path "secret/data/*" {
  capabilities = ["read"]
}
EOF

# Bind the policy to a Kubernetes service account
vault write auth/kubernetes/role/eso \
  bound_service_account_names=external-secrets \
  bound_service_account_namespaces=external-secrets \
  policies=eso-read \
  ttl=24h
```

---

## External Secrets Operator (ESO)

[External Secrets Operator](https://external-secrets.io/) is the preferred
secrets integration for OpenPrime. It pulls secrets from Vault into Kubernetes
`Secret` objects.

```yaml
# ClusterSecretStore pointing at the Vault cluster
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "https://vault.example.internal:8200"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "eso"
          serviceAccountRef:
            name: "external-secrets"
            namespace: "external-secrets"
      caBundle: "<base64-encoded CA cert>"
```

The `caBundle` is the CA from `vault_tls_ca_file` (generated or BYO), base64-encoded.

---

## Vault Agent

For workloads that need dynamic secrets injected as files (rather than Kubernetes
`Secret` objects), deploy the Vault Agent Injector. The injector authenticates via
the same Kubernetes auth method.

```yaml
# ArgoCD Application example
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: vault-agent-injector
  namespace: argocd
spec:
  source:
    repoURL: https://helm.releases.hashicorp.com
    chart: vault
    targetRevision: "0.28.0"
    helm:
      values: |
        injector:
          enabled: true
        server:
          enabled: false
        global:
          externalVaultAddr: "https://vault.example.internal:8200"
```

---

## Post-role workflow

After the role completes successfully:

1. **Recover the init file** from the Ansible controller:
   ```bash
   ansible-vault view .vault_init_<cluster_id>.json --vault-password-file ~/.vault-pass
   ```

2. **Store unseal/recovery keys** in a secure location (hardware token, sealed
   envelope procedure, or a separate secrets manager). Never leave them only on the
   controller disk.

3. **Rotate the root token:** Vault's root token should be revoked once initial
   configuration is complete and a proper admin policy + token is in place:
   ```bash
   vault token revoke <root_token>
   ```

4. **Configure auth methods and secrets engines** appropriate for your workload
   (Kubernetes auth, PKI secrets engine, KV v2, etc.).

5. **Test failover:** stop `vault.service` on the leader node and confirm one of
   the standby nodes is promoted to active within the `autopilot.min_quorum_size`
   window.
