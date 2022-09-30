# TEST FARM

An ansible tool to provision an host with VMs of different machine, architecture and OS configs and to provision VMs with custom tasks
## Requirements
- Packages
  - `python` >= 2.6
  - `python3-libvirt`
    - required by community.libvirt.virt  
  - `python3-lxml`
    - required by community.libvirt.virt_net
  - `sshpass`
    - optional but required to use password on ssh on vm connections
    - otherwise use [ansible vault](https://docs.ansible.com/ansible/2.8/user_guide/vault.html)
  - `libvirt-bin`
    - required by `guest_provision` role to handle snapshots using virsh
  - `ip`
    - required to detect VM ip through an interface if not provided
  - [unarchive](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/unarchive_module.html) module dependencies
  - `gzip`, `bunzip2`
    - required if using unsupported archive format by the unarchive module
- System running hypervisor:
  - Supported platform:
    - Theoretically any GNU/Linux distribution
    - Tested:
      - debian
  - hypervisor
    - default: `qemu`
  - libvirt environment
    - libvirt daemon active and running
- User:
  - group: `libvirt_group` var value (default: `libvirt`)
    - require to have access to libvirt features
      - See your distribution requirements to use libvirt features
  - group: `kvm`
    - require to use kvm device
  - group: `hypervisor_group` var value (default: `libvirt-qemu` )
    - require to change files ownership to allow the hypervisor to access to
      - in general see your hypervisor requirements
  - Note: default `hypervisor_group` and `libvirt_group` vars are defined in `roles/kvm_provision/defaults/main.yaml`, so they can be overridden on hypervisor's host (group)vars according to your use case.

## Usage

If you want use custom ansible config, you can use `ANSIBLE_CONFIG="./ansible.cfg"` first.
### KVM host provisioning: Install and setup VMs on KVM hosts
```
ansible-playbook main.yaml -K
```
By default the VMs configurations are created from the definitions provided into `vars/vm_definitions.yaml`.
### KVM guest provisioning: Install dependencies on VM guests and run the guest lifecycle phases's
```
ansible-playbook guest_provision.yaml
```
By default the VMs info use the configurations parsed using `roles/parse_vms_definitions` and from thee definitions provided into `vars/vm_definitions.yaml`.


## The VM Guest provisioning
### Prerequisite

After you ran the `kvm_provision` role, before start the VM guest provisioning you have to run the `guest_provision` role first to include the VM you want to process and add them them as ansible hosts with a dynamic inventory with the following role utility:
```
- name: Add into Virtual Machines lifecycle
  loop: "{{ virtual_machines }}"

  include_role:
    name: guest_provision
  vars:
    vm: "{{ vm }}"
    should_add_vm: True

  loop_control:
    loop_var: vm
```
After that
- each VM host is added as ansible host
- the `VM definition` is added as global `vm`  host var in VM's lifecycle.
- the hypervisor's inventory host is added as global `kvm_host` host var in VM's lifecycle.
- Each VM host are added to the following ansible groups:
  - `vms`
  - `"{{ vm.metadata.name }}"`
  - `"{{ vm.metadata.platform_name }}"`
  - `"{{ vm.metadata.arch_name }}"`

In your playbook you can start the guest provisioning using:
```
- name: "Start Guest provisioning"
  hosts: vms # or any group you want 
  serial: 1
  tasks:
  - include_role:
      name: guest_provision
    vars:
      should_run_guest_main: True
```
### The VM Guest lifecycle
1. **dependencies** phase
   - Create 'clean' snapshot
   - Install use case dependencies
     - Import var dependencies if any (`vars/phases[/import_path]/dependencies.yaml`)
       - Install defined dependencies ( uses [ansible.builtin.package module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/package_module.html))
     - Run dependencies tasks (`tasks/phases[/import_path]/dependencies.yaml`)
   - Create 'dependencies' snapshot
2. **Init** use case phase: 
   - Run init tasks `tasks/phases[/import_path]/init.yaml`
   - Create 'init' snapshot
3. **Main** use case phase: 
   - Run main tasks `tasks/phases[/import_path]/main.yaml`
4. **Terminate** use case phase: 
   - Run end tasks `tasks/phases[/import_path]/terminate.yaml`

Where `import_path` is optional and it's a subpath of 2 nested folders if the use case needs specific tasks/vars for a target on platform or only platform; for instance:
- *debian_11* folder (`vm.metadata.platform_name` value in `platforms/debian_sid.yml`)
    - *amd64* folder (`vm.metadata.arch_name` value in `targets/amd64.yml`)
      - tasks or vars files, ... specific for *amd64* targets in *debian_11* platforms
    - *arm64* folder (`vm.metadata.arch_name` value in `targets/arm64.yml`)
      - tasks or vars files, ... specific for *arm64* targets in *debian_11* platforms
- *fedora_36* folder (`vm.metadata.platform_name` value in `platforms/fedora_36.yml`)
    - *amd64* folder (`vm.metadata.arch_name` value in `targets/amd64.yml`)
      - tasks or vars files, ... specific for *amd64* targets in *fedora_36* platforms
    - tasks or vars files, ... specific *fedora_36* platforms but any target
- tasks or vars files, ... generic for any platform and target which file does not exists with a specific `import_path` sub path

This is useful when some dependencies have different alias in some platform's packets manager, or user needs "ad hoc" tasks/vars for some others use cases.

