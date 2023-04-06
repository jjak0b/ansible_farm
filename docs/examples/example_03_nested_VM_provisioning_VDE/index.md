# Nested VM provisioning with User Networking and VDE

## Intro

This example will show how to deploy a VM on a bare metal hypervisor host and use this VM as hypervisor to create 2 others nested VMs: one of these uses user networking and the other one will be connected to its hypervisor through [VDE](http://wiki.virtualsquare.org/#!repos.md#VDE).

- [Full code](//github.com/jjak0b/ansible_farm/tree/master/docs/examples/example_03_nested_VM_provisioning_VDE/)

The following document will use the following terms:

- **L0** hypervisor is the real bare metal hypervisor host
- **L1** VM (hypervisor) is a VM, which hypervisor is a L0 hypervisor
- **L2** VM is a nested VM which hypervisor is a L1 VM

## Prerequisite

- [kvm_provision](../../roles/kvm_provision.md#Requirements ) role requirements
- [guest_provision](../../roles/guest_provision.md#Requirements ) role requirements
- ``` ansible-galaxy install -r ./requirements.yml ``` 
- Recommended: Enable [Nested KVM](https://www.linux-kvm.org/page/Nested_Guests) on your bare metal host

## Define the main playbook

Let's define the playbook [main.yaml](main.yaml) in same way of the [example_02](../example_02_VM_provisioning/index.md) but lets divide it in 3 playbooks:

- `init_vm_definitions.yaml`
  ```
    # This playbook can use all variables from parse_vms_definitions and init_vm_connection roles defined on each hypervisor's specific inventory

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

## Define the inventory

Let's define an inventory such that:

- The L0 (bare metal) hypervisor will deploy an L1 VM of platform **debian_vs**
- The L1 (VM) hypervisor will deploy these L2 VMs of platforms:

  - **debian_vs_vde**
  - **debian_vs_user**

Note: L1 hypervisors will auto-generate and setup the required ssh jump hosts ( through `init_vm_connection` role ) by using the `-J` parameter to allow ansible to `ssh` into VMs. In specific  the following variable will be set as: ``` ansible_ssh_common_args: "-J {{ generated_jumphosts | join(',') }} -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no" ``` where the generated jump hosts are the nested `kvm_host` hypervisors formated as `ansible_user@ansible_host:ansible_port`

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
      # L1 deploy these VMs
      config:
        permutations:
          targets:
            - "amd64"
          platforms: 
            - "debian_vs_vde"
            - "debian_vs_user"

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

## The VMs provisioning

The **1st** `run_vm_provision.yaml` playbook will deploy and run the provisioning of the L1 VM about the **debian_vs** platform which VM provisioning phases will run tasks such that:

  - In **debian_vs**'s init phases:

    - Upgrade preinstalled packages
    - Install [this collection's requirements](../../index.md#Requirements) such as libvirt env and dependencies
    - Install a `default` libvirt pool by using the external [stackhpc.libvirt-host](https://github.com/stackhpc/ansible-role-libvirt-host) utility role
  
  - In **debian_vs**'s main phase

    - Install `vde2`
    - Import a `vde_provisioning` local utility role, in short:

      - Create a tap interface to connect through VDE:
        ```
        ip tuntap add mode tap name tap0 user {{ ansible_user }}"
        ip addr add 10.0.0.254/24 dev tap0
        ip link set tap0 up
        ```
      - Plugs the tap interface to the specified `vde_network: vxvde://234.0.0.1` in background
        ```
        vde_plug tap://tap0 {{ vde_network }} -p {{ pidfile_path }} &
        ```
    
    - Init the L2 VMs with `parse_vms_definitions` and `init_vm_connection`, adding it-self as their hypervisor

  - In **debian_vs**'s terminate phase
    - Remove the `shutdown` phase to allow ansible to run the VM provisioning process on next playbook.
  
  The **2nd** `run_vm_provision.yaml` playbook will deploy and run the provisioning of the nested L2 VMs which VM provisioning phases won't run any meaningful phase expect for the `terminate` phase:
  - The `terminate` will _notify_ the `shutdown_hypervisor` handler (defined in `guest_provision` role) which will trigger externally the `shutdown` phase of the **L1** hypervisor through **L0** if **L1** is an already tracked VM on same ansible instance at the end of the current playbook.

## Run

```
ANSIBLE_CONFIG=ansible.cfg ansible-playbook main.yaml
```

You shouldn't get any error and both L2 VMs will result reachable by using the ssh jump host feature: the **debian_vs_vde** VM through a VDE network and **debian_vs_user** through the QEMU slirp user networking.

Warning: If you are trying to connect to your VM with a VDE network and its hypervisors is Ubuntu 22.04 LTS (and maybe future releases) and you get the [error](https://askubuntu.com/questions/1427364/qemu-vde-network-backend-error) `Parameter 'type' expects a netdev backend type` you have to build QEMU binaries from source and use it in your VM definition to use VDE.
