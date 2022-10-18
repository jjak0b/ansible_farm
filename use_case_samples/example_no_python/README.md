
Use case
--------
this example shows how to connect to a VM which require manual setup of the image and doesn't have python installed


Usage
-----

### Hypervisor and VM provisioning
```
ansible-playbook main.yaml
```

### Hypervisor provisioning only
```
ansible-playbook main.yaml --skip-tags "guest_provision"
```

### VM provisioning only
```
ansible-playbook main.yaml --skip-tags "kvm_provision"
```
