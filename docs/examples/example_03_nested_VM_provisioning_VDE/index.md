Nested VM provisioning with User Networking and VDE
========================================================

Intro
-----

This example will show how to deploy a VM on a bare metal hypervisor host and use this VM as hypervisor to create 2 others nested VMs: one of these uses user networking and the other one will be connected to its hypervisors through [VDE](http://wiki.virtualsquare.org/#!repos.md#VDE).

- [Full code](//github.com/jjak0b/test_farm/tree/master/docs/examples/example_03_nested_VM_provisioning_VDE/)


Prerequisite
-------------

- [kvm_provision](../../roles/kvm_provision.md#Requirements ) role requirements
- [guest_provision](../../roles/guest_provision.md#Requirements ) role requirements
- ``` ansible-galaxy install -r ./requirements.yml ``` 
- Recommended: Enable [Nested KVM](https://www.linux-kvm.org/page/Nested_Guests) on your bare metal host

Define the main playbook
------------------------

Let's define the playbook [main.yaml](main.yaml) in same way of the [example_02](../example_02_VM_provisioning/index.md) but lets divide it in 3 playbooks:

- `init_vm_definitions.yaml`
  ```
    # This playbook can use all variables from parse_vms_definitions and init_vm_connection roles defined on each hypervisor's specific inventory
    ---

    - name: Init VM on Hypervisor
    hosts: "{{ hypervisors_group | default('hypervisors') }}"
    gather_facts: yes
    roles:
        - parse_vms_definitions
    tasks:  

        - name: init VM connection
        loop: "{{ virtual_machines }}"

        include_role:
            name: init_vm_connection
        
        loop_control:
            loop_var: vm
  ```

- `run_vm_provision.yaml`
  ```
  # This playbook can use all variables from kvm_provision and guest_provision roles
  ---

  - name: VM provisioning on Hypervisor host
  hosts: "{{ vms_group | default('vms') }}"
  gather_facts: no
  serial: 1
  tasks:

    - block:
      
        - name: gather min facts of hypervisor host since some definitions require to use them
          setup:
            gather_subset: 
            - '!all'
        
        - name: "start KVM Provision role for '{{ vm.metadata.name }}'"
          include_role: 
            name: kvm_provision
        
      delegate_to: "{{ kvm_host }}"
      
      tags:
        - kvm_provision

    - block:

        - name: "Start VM provisioning of '{{ vm.metadata.name }}' "
          include_role: 
            name: guest_provision
        
        - name: Flag VM as processed
          add_host:
            name: "{{ inventory_hostname }}"
            groups:
              - processed
      tags:
        - guest_provision

  ```
- `main.yaml`
  ```
  - name: "Assign VM to L0 hypervisors"
    import_playbook: init_vm_definitions.yaml
    vars:
      hypervisors_group: hypervisors

  - name: Create and provision each L1 VM on its assigned L0 hypervisor
    import_playbook: run_vm_provision.yaml
    vars:
      vms_group: vms

  - name: Create and provision each L2 VM on its assigned L1 hypervisor
    import_playbook: run_vm_provision.yaml
    vars:
      vms_group: vms:!processed
  ```

Note:  `run_vm_provision.yaml` is repeated twice since the changes to the L1 hypervisor's inventory are made on runtime such that at the end of all L1 VM provisioning, then it will start the L2 VM provisioning by running the 2nd `run_vm_provision.yaml` playbook.

Define the inventory
----

Let's define an inventory such that:
- The L0 (bare metal) hypervisor will deploy an L1 VM of platform **debian_vs**
- The L1 (VM) hypervisor will deploy these L2 VMs of platforms:
    - **debian_vs_vde**
    - **debian_vs_user**

Note: L1 hypervisors will auto-generate and setup the required ssh jump hosts ( through `init_vm_connection` role ) by using the `-J` parameter to allow ansible to `ssh` into VMs. In specific  the following variabl will be set: ``` ansible_ssh_common_args: "-J {{ generated_jumphosts | join(',') }} -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no" ``` where the generated jump hosts are the nested `kvm_host` hypervisors formated as `ansible_user@ansible_host:ansible_port`

Warning: The `-J` can only use ssh keys to authenticate, so you need to provided at least the `ansible_ssh_private_key_file` for your L0 hypervisors in the inventory. You can set it as `~/.ssh/id_rsa` and generate it with ```ssh-keygen -b 2048 -t rsa```. You should specify also the keys used by the VMs and inject them into the used image but if you use the `callbacks/sources/setup_image.yaml` callback-task, then will do it for you.

```
all:
  hosts:
    L0-hypervisor:
      ansible_connection: ssh
      ansible_user: user
      ansible_host: localhost
      ansible_become_method: enable
      ansible_ssh_private_key_file: ~/.ssh/id_rsa
      # L0 deploy these VMs
      config:
        permutations:
          targets:
            - "amd64"
          platforms:
            - "debian_vs"
    VM debian_vs_amd64:
      ansible_port: 2201
      # L1 deploy these VMs
      config:
        permutations:
          targets:
            - "amd64"
          platforms: 
            - "debian_vs_vde"
            - "debian_vs_user"

    VM debian_vs_user_amd64:
      ansible_port: 2202
  children:
    hypervisors:
      hosts:
        L0-hypervisor:
      # This will be added later by the guest provision phases
      # VM debian_vs_amd64
    vms:
      vars:
        ansible_connection: ssh
        project_id: example_03
        project_revision: 0
        ansible_ssh_private_key_file: ~/.ssh/id_ssh_rsa_vm
  vars:
    vde_network: vxvde://234.0.0.1
    
```

The VMs provisioning
--

- Let's define the VM provisioning phases for the **debian_vs** platform such that:
  - In **debian_vs**'s init phases:
    - Upgrade preinstalled packages
    - Install [this collection's requirements](../../index.md#Requirements) such as libvirt env and dependencies
    - Install a `default` libvirt pool by using the external [stackhpc.libvirt-host](https://github.com/stackhpc/ansible-role-libvirt-host) utility role
  - In **debian_vs**'s main phase
    - Init the VM provisioning on the nested VMs, adding it-self as their hypervisor

  - in 2nd `run_vm_provision.yaml` playbook: will Deploy and provisioning of the nested L2 VMs.

Test it
--

```
cd docs/examples/example_03_nested_VM_provisioning_VDE
ANSIBLE_CONFIG=ansible.cfg ansible-playbook main.yaml
```

