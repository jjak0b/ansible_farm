Deploy a VM as hypervisor
========================================================

Intro
-----

In this example we are going to deploy a VM on a hypervisor host and use this VM as hypervisor to create nested VMs.

- [Full code](//github.com/jjak0b/test_farm/tree/master/docs/examples/example_deploy_libvirt_vm/)

Prerequisite
-------------

- [kvm_provision](../../roles/kvm_provision.md#Requirements ) role requirements
- [guest_provision](../../roles/guest_provision.md#Requirements ) role requirements
- ``` ansible-galaxy install -r ./requirements.yml ``` 

Define the main playbook
------------------------

Let's define the playbook [main.yaml](main.yaml) in same way of way of the [example_02](../example_02_VM_provisioning/index.md).

Define the inventory
----
Let's define an inventory such that:
- Our hypervisor will deploy a the vm of platform **vm-as-hypervisor**
- The **VM vm-as-hypervisor_amd64** will deploy the VM **debian_vs** when becomes an hypervisor

```
all:
  children:
    hypervisors:
      hosts:
        local_hypervisor:
          ansible_connection: local
          ansible_host: localhost
      vars:
        config:
          permutations:
            targets:
              - "amd64"
            platforms:
              - "vm-as-hypervisor"
    vm-as-hypervisor:
      hosts:
        VM vm-as-hypervisor_amd64:
      vars:
        config:
          permutations:
            targets:
              - "amd64"
            platforms:
              - "debian_vs"
    
```

The VMs provisioning
--

- Let's define the VM [provisioning phases](provisioning_phases/vm-as-hypervisor) for the platform **vm-as-hypervisor** such that
  - Upgrade preinstalled packages
  - Install [this collection's requirements](../../index.md#Requirements) such as libvirt env and dependencies
  - Install a default libvirt pool by using the external [stackhpc.libvirt-host](https://github.com/stackhpc/ansible-role-libvirt-host) utility role
  - Adds it self as hypervisor host and start to deploy a (nested) **debian_vs** VM on it self
  - Starts the guest provisioning on the nexted VM

Note: The guest provisioning of this example will fail on the nested VM because will result unreachable. To make it to work We need to setup properly the connection plugin and/or network to allow ansible to connect to the nested VM; for example we may use ssh host jump but we won't cover it in this demo.

