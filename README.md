# TEST FARM

run kvm provisioning:

```
ANSIBLE_CONFIG="./ansible.cfg" \
ansible-playbook main.yaml -K
```

ANSIBLE_CONFIG="./ansible.cfg" \
ansible-playbook guest_provision.yaml
