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

- The L0 (bare metal) hypervisor will deploy an L1 VM of platform **debian_sid**
- The L1 (VM) hypervisor will deploy these L2 VMs of platforms:

  - **debian_sid_vde**
  - **debian_sid_user**

Note: L1 hypervisors will auto-generate and setup the required ssh jump hosts ( through `init_vm_connection` role ) by using the `-F` parameter of the `ssh` command to allow ansible to authenticate through `ssh` into VMs using a custom auto-generated configuration file with all proxy jumps which are the nested `kvm_host` hypervisors

Warning: You need to provide at least the `ansible_ssh_private_key_file` for your L0 hypervisors in the inventory. You can set it as `~/.ssh/id_rsa` and generate it with ```ssh-keygen -b 2048 -t rsa``` command. You should specify also the keys used by the VMs and inject them into the used image but if you use the `callbacks/sources/setup_image.yaml` callback-task, then it will do it for you.

```
all:
  hosts:
    L0-hypervisor:
      ansible_connection: local
      ansible_host: localhost
      ansible_ssh_private_key_file: ~/.ssh/id_rsa
      # L0 deploy these VMs
      config:
        permutations:
          targets:
            - "amd64"
          platforms:
            - "debian_sid"
    VM debian_sid_amd64:
      # L1 deploy these VMs
      config:
        permutations:
          targets:
            - "amd64"
          platforms: 
            - "debian_sid_vde"
            - "debian_sid_user"

  children:
    hypervisors:
      hosts:
        L0-hypervisor:
      # This will be added later by the guest provision phases
      # VM debian_sid_amd64
    vms:
      vars:
        # increase timeout since L2 VMs ma be very slow
        connection_timeout: 200
        ansible_connection: ssh
        project_id: example_03
        project_revision: 0
        ansible_ssh_private_key_file: ~/.ssh/id_ssh_rsa_vm
  vars:
    vde_network: vxvde://234.0.0.1
    
```

## Define the platforms

Lets define a parametric `_debian.yaml` platform that will be used to define common data for other platforms:

```
- name: Defining platform with usermode networking
  vars:
    # version_name: bookworm
    # version_major_num: 12
    # version: "20230330-1335"
    period: daily
    vm_name: "VM {{ vm.metadata.platform_name }}_{{ vm.metadata.target_name }}"
    debian_arch_names:
      "arm64": "arm64"
      "amd64": "amd64"
      "ppc64": "ppc64el"
  block:

    - vars:
        ssh_forward_port: "{{ hostvars[ vm_name ].ansible_port | default( 65535 | random( start=2201, seed=vm_name), true ) }}"
      block:
      
        - name: Set connection port for VM
          add_host:
            name: "{{ vm_name }}"
            ansible_port: "{{ ssh_forward_port }}"
      
        - name: Define platform definition
          vars:
            uri_base: "https://cdimage.debian.org/cdimage/cloud/{{ version_name }}/{{ period }}/{{ version }}"
            image_name: "debian-{{ version_major_num }}-nocloud-{{ debian_arch_names[ vm.metadata.target_name ] }}-daily-{{ version }}.qcow2"
          set_fact:
            vm:
              # metadata properties are used inside a task for installation and setup purposes
              metadata:
                name: "{{ vm_name }}"
                connection: "qemu:///session"
                auth:
                  become_user: "root"
                  become_password: &root_pass "virtualsquare"
                  become_method: su
                  user: "user"
                  password: *root_pass
                sources:
                - before_provision:
                  - callback: callbacks/sources/fetch.yaml
                    src: "{{ uri_base }}/{{ image_name }}"
                    dest: "{{ image_name }}"
                  - callback: callbacks/sources/copy.yaml
                    src: "{{ image_name }}"
                    dest: &tmp_image_file_name "tmp-{{ vm.metadata.platform_name }}_{{ vm.metadata.target_name }}_{{ image_name }}"
                  - callback: callbacks/sources/setup_image.yaml
                    src: *tmp_image_file_name
                    keys_path: "{{ vm.metadata.tmp_dir }}id_ssh_rsa_vm"
                  on_provision:
                    src: *tmp_image_file_name
                    dest: &image_file_name "{{ vm.metadata.platform_name }}_{{ vm.metadata.target_name }}_{{ image_name }}"
              vcpus: 2
              ram: 1536
              disks:
                - type: "qcow2"
                  devname: "hda"
                  src: *image_file_name
              net:
                type: user
                source: "hostfwd=tcp:127.0.0.1:{{ ssh_forward_port }}-:22"
                mac: "{{ '52:54:00' | random_mac( seed = vm_name ) }}"
                # note: user networking is associated to hypervisor's ip
                ip: "localhost"

```

As you can see the  `setup_image` callback-task prepare the image and inject the SSH authorized key.

Note: the `ansible_ssh_private_key_file: ~/.ssh/id_ssh_rsa_vm` will be generated by the `setup_image.yaml` callback-task and used later as authorized key and we will use the same key to authenticate into the virtual machines automatically.

Lets define the real platforms:

- **debian_sid** will be a L1 VM hypervisor that will use a resized debian sid image to store the hypervisor requirements L2 images
  ```

  - block:
      - import_tasks: _debian.yaml
    vars:
      version_name: sid
      version_major_num: sid
      version: '20230403-1339'
    
  - vars:
      new_task:
        callback: callbacks/sources/extend_and_convert.yaml
        src: "{{ vm.metadata.sources[0].before_provision[2].src }}"
        dest: "{{ vm.metadata.sources[0].before_provision[2].src }}"
        from_format: qcow2
        to_format: qcow2
        partition_number: 1
        size: 10G
    block:

      - name: Add new task to imported definition
        set_fact:
          vm: "{{ vm | combine(vm_override, recursive=true ) }}"
        vars:
          vm_override:
            metadata:
              sources:
                - before_provision: "{{ vm.metadata.sources[0].before_provision[:2] + [ new_task ] + vm.metadata.sources[0].before_provision[2:] }}"
                  on_provision: "{{ vm.metadata.sources[0].on_provision }}"
  ```
  - The `extend_and_convert.yaml` callback-task is used to resize the debian image, similar to how the `debian_vs` image [was created](https://github.com/virtualsquare/freshly_brewed_virtualsquare/blob/master/brew_v2)

- **debian_sid_user** will be a L2 VM with network type 'user' configured, so just reuse the *debian* platform:

  ```
  - block:
      - import_tasks:
          file: _debian.yaml
    vars:
      version_name: sid
      version_major_num: sid
      version: '20230403-1339'

  - name: Override platform
    block:
      - name: Override for user network setup
        set_fact:
          vm: "{{ vm | combine( vm_override, recursive=true) }}"
        vars:
          vm_override:
            ram: 512
  ```

- **debian_sid_vde** will be a L2 VM connected with a 'vde' network type, so we need to add an callback-task to insert a network interface configuration file into the image such that the VM will be reachable through its interface address:
  ```
  - block:
      - import_tasks:
          file: _debian.yaml
    vars:
      version_name: sid
      version_major_num: sid
      version: '20230403-1339'

  - name: Reset connection port for VM
    add_host:
      name: "{{ vm.metadata.name }}"
      ansible_port: 22

  - name: Override platform
    vars: 
      new_task:
        callback: callbacks/sources/setup_connection.yaml
        src: "{{ vm.metadata.sources[0].on_provision.src }}"
    block:
      - name: Override for VDE setup
        set_fact:
          vm: "{{ vm | combine( vm_override, recursive=true) }}"
        vars:
          vm_override:
            metadata:
              # reuse the old_before_provision but insert the network preprocessing before the main setup image
              sources:
                # Note: For some strange reason if we add the "setup network" as last callback task, the VM won't start ssh service
                # maybe because the "setup image" callback-task uses 'ssh-keygen -A'
                - before_provision: "{{ vm.metadata.sources[0].before_provision[:2] + [ new_task ] + vm.metadata.sources[0].before_provision[2:] }}"
                  on_provision: "{{ vm.metadata.sources[0].on_provision }}"
            ram: 512
            net:
              type: vde
              source: "{{ vde_network }}"
              ip: "10.0.0.21"
              mask: "24"
              gateway: "10.0.0.254"
  ```

## The VMs provisioning

The **1st** `run_vm_provision.yaml` playbook will deploy and run the provisioning of the L1 VM about the **debian_sid** platform which VM provisioning phases will run tasks such that:

  - In **debian_sid**'s init phases:

    - Upgrade preinstalled packages
    - Install [this collection's requirements](../../index.md#Requirements) such as libvirt env and dependencies
    - Install a `default` libvirt pool by using the external [stackhpc.libvirt-host](https://github.com/stackhpc/ansible-role-libvirt-host) utility role
  
  - In **debian_sid**'s main phase

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

  - In **debian_sid**'s terminate phase
    - Remove the `shutdown` phase to allow ansible to run the VM provisioning process on next playbook.
  
  The **2nd** `run_vm_provision.yaml` playbook will deploy and run the provisioning of the nested L2 VMs which VM provisioning phases won't run any meaningful phase expect for the `terminate` phase:
  - The `terminate` will _notify_ the `shutdown_hypervisor` handler (defined in `guest_provision` role) which will trigger externally the `shutdown` phase of the **L1** hypervisor through **L0** if **L1** is an already tracked VM on same ansible instance at the end of the current playbook.

## Run

```
ANSIBLE_CONFIG=ansible.cfg ansible-playbook main.yaml
```

You shouldn't get any error and both L2 VMs will result reachable by using the ssh jump host feature: the **debian_sid_vde** VM through a VDE network and **debian_sid_user** through the QEMU slirp user networking.

Warning: If you are trying to connect to your VM with a VDE network and its hypervisors is Ubuntu 22.04 LTS (and maybe future releases) and you get the [error](https://askubuntu.com/questions/1427364/qemu-vde-network-backend-error) `Parameter 'type' expects a netdev backend type` you have to build QEMU binaries from source and use it in your VM definition to use VDE.
