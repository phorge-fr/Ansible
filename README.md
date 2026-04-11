# Ansible

![Phorge logo](https://avatars.githubusercontent.com/u/187407936?s=200&v=4)

This repository stores all the ansible configurations & file required to run the [Phorge Cloud Infrastructure](https://phorge.fr).

## Setup

requirements:

- [Ansible](https://ansible.com)

1. Install the requirements

```bash
ansible-galaxy install -r requirements.yml
```

2. Test environment

```bash
ansible -m ping all
```

## Encryption and execution

### Encrypt variables

```bash
ansible-vault encrypt_string --vault-password-file vault_pass '<yaml_value_to_encrypt>' --name '<yaml_key>'
```

### Encrypt files

```bash
ansible-vault encrypt_string \
  --vault-password-file vault_pass \
  --stdin-name '<yaml_key>' \
  < file_to.encrypt
```

### Run playbooks with encrypted variables

```bash
ansible-playbook -i inventories/production/hosts playbooks/<playbook>.yml --vault-password-file vault_pass 
```