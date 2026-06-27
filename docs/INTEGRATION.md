# Integration Guide

How to consume the `devopsgroupeu.hashicorp-vault` role in other projects and how
to wire the Vault cluster it produces into your platform as a secrets backend.

---

## Table of Contents

- [Consuming the role via requirements.yml](#consuming-the-role-via-requirementsyml)
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

## Vault as a secrets backend for Kubernetes

After the Vault role completes, configure the Kubernetes authentication method:

```bash
# Enable the Kubernetes auth method
vault auth enable kubernetes

# Configure it with the Kubernetes cluster endpoint
vault write auth/kubernetes/config \
  kubernetes_host="https://$(kubectl config view --raw -o json | jq -r '.clusters[0].cluster.server')" \
  kubernetes_ca_cert=@/path/to/kubernetes-ca.crt  # your cluster's CA cert

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

[External Secrets Operator](https://external-secrets.io/) is a common
secrets integration for Kubernetes. It pulls secrets from Vault into Kubernetes
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
