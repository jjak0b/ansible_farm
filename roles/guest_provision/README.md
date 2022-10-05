Role Name
=========

This role provision VMs with custom VM lifecycle phases organized in snapshotted phases

### The VM Guest lifecycle
1. **dependencies** phase
   - Create 'clean' snapshot
   - Install use case dependencies
     - Run dependencies tasks (`{{ import_path }}/dependencies.yaml`)
   - Create 'dependencies' snapshot
2. **Init** use case phase: 
   - Run init tasks `{{ import_path }}/init.yaml`
   - Create 'init' snapshot
3. **Main** use case phase: 
   - Run main tasks `{{ import_path }}/main.yaml`
4. **Terminate** use case phase: 
   - Run end tasks `{{ import_path }}/terminate.yaml`
 - 
Where `import_path` is a subpath that match with the most detailited phase file location, according to the target and platform type of the VM.
The `import_path` is the one in the following priority list path which contains a phase file:
- `"{{ ( phases_lookup_dir_path, vm.metadata.platform_name, vm.metadata.arch_name) | path_join }}"`
- `"{{ ( phases_lookup_dir_path, vm.metadata.platform_name ) | path_join }}"`
- `"{{ phases_lookup_dir_path }}"`

A use case may needs specific tasks/vars for a target on platform or only platform; for instance:
- *debian_11* folder (`vm.metadata.platform_name` value in `platforms/debian_sid.yml`)
    - *amd64* folder (`vm.metadata.arch_name` value )
      - tasks or vars files, ... specific for *amd64* targets in *debian_11* platforms
    - *arm64* folder (`vm.metadata.arch_name` value )
      - tasks or vars files, ... specific for *arm64* targets in *debian_11* platforms
- *fedora_36* folder (`vm.metadata.platform_name` value )
    - *amd64* folder (`vm.metadata.arch_name` value )
      - tasks or vars files, ... specific for *amd64* targets in *fedora_36* platforms
    - tasks or vars files, ... specific *fedora_36* platforms but any target
- tasks or vars files, ... generic for any platform and target which file does not exists with a specific `import_path` sub path

The `import_path` is useful when some dependencies have different alias in some platform's packets manager, or user needs "ad hoc" tasks/vars for some others use cases.

Requirements
------------
- System running hypervisor:
  - Supported platform:
    - Theoretically any GNU/Linux distribution
    - Tested:
      - debian
  - libvirt environment
    - libvirt daemon active and running
- Packages
  - `sshpass`
    - optional but required to use password on ssh on vm connections
    - otherwise use [ansible vault](https://docs.ansible.com/ansible/2.8/user_guide/vault.html)
  - `libvirt-bin`
    - required by `guest_provision` role to handle snapshots using virsh

Role Variables
--------------

- `kvm_host`
  - required
  - it's the hypervisor's inventory_name that allow to delegate some tasks related to the hypervisor
- `vm`
  - required
  - It's the `VM definition` object required to specify which VM have to be provisioned
- `ansible_connection`
  - recommended
  - standard connection plugins are supported
  - `community.libvirt.libvirt_qemu` is supported

Dependencies
------------

- Format of `VM definition` in `vm` var required by `roles/kvm_provision` and used in this role phase tasks
- Format of `VM definition` in `vm` var produced by `roles/parse_vms_provision`'s output and used in this role phase tasks

Example Playbook
----------------
```
- name: VM provisioning on Hypervisor host
  hosts: vms
  gather_facts: no
  serial: 1
  tasks:
  - block:
    - name: gather min facts if definitions use them
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
    tags:
    - guest_provision
```

License
-------

BSD

Author Information
------------------

None
