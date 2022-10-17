Usage
-----

### Hypervisor and VM provisioning
```
ansible-playbook main.yaml -K
```
The `-K` may be required when using `roles/kvm_provision` because `setup_hypervisor/prerequisite/targets/<target_name>.yml` try to install emulator dependencies on hypervisor host

### Hypervisor provisioning only
```
ansible-playbook main.yaml --skip-tags "guest_provision" -K
```

### VM provisioning only
```
ansible-playbook main.yaml --skip-tags "kvm_provision"
```
