# Ansible Role HashiCorp Vault
 
 Example Playbook:
 
 ```yaml
- name: Create temporary certificates
  hosts: vault_nodes
  vars_files:
    - group_vars/hashicorp_vault.yml
  roles:
    - role: devopsgroup.hashicorp_vault
 ```
 