# jjak0b.deploy_farm Collection

An ansible tool to provision an host with VMs of different machine, architecture and OS configs and to provision VMs with custom tasks

The Hypervisor and VM provisioning
----------------------------------

the VMs configurations should be defined in a var file, for example `vms_config.yaml` and these configurations vars should be then provided as input of `roles/parse_vms_definitions` such that generates `VM definitions` items in a `virtual_machines` list.

### Usage 
- For each host of hypervisors
  - For each `vm` item ( for example of `virtual_machines` list ) should be provided as input of:
    - `roles/init_vm_connection` to add a new ansible host entry to the inventory and define a libvirt network and a DHCP entry for connection such that the vm should be connected to it
      - After that
        - each VM host is added as ansible host
        - the `VM definition` is added as global `vm`  host var in VM's lifecycle.
        - the hypervisor's inventory host is added as global `kvm_host` host var in VM's lifecycle.
        - Each VM host are added to the following ansible groups:
          - `vms`
          - `"{{ vm.metadata.name }}"`
          - `"{{ vm.metadata.platform_name }}"`
          - `"{{ vm.metadata.target_name }}"`
    - `roles/kvm_provision` to define and install VM resources 

- For each host in `vms` should run:
  - ( alternatively use `roles/kvm_provision` here to define and install VM resources if `vm` var as `VM definition` object is defined as vm host variable )
  - `roles/guest_provision` to provision the VM with the guest lifecycle

The VM Guest lifecycle
----------------------
The lifecycle of the provisioned VM runs the following phases:

0. **Init** use case phase
   1. Restore to '**init**' snapshot (if exists)
   2. otherwise restore or create the '**clean**' snapshot
      1. **dependencies** phase
         - Run dependencies tasks (`{{ import_path }}/dependencies.yaml`)
      2. use case phase: 
         - Run init tasks `{{ import_path }}/init.yaml`
         - Create 'init' snapshot
1. **Main** use case phase: 
   - Run main tasks `{{ import_path }}/main.yaml`
2. **Terminate** use case phase: 
   - Run end tasks `{{ import_path }}/terminate.yaml`

Where `import_path` is a subpath that match with the most detailited phase file location, according to the target and platform type of the VM.
The `import_path` is the one in the following priority list path which contains a phase file:
- `"{{ ( phases_lookup_dir_path, vm.metadata.platform_name, vm.metadata.target_name| path_join }}"`
- `"{{ ( phases_lookup_dir_path, vm.metadata.platform_name ) | path_join }}"`
- `"{{ phases_lookup_dir_path }}"`

A use case may needs specific tasks/vars for a target on platform or only platform; for instance:

- *debian_11* folder (`vm.metadata.platform_name` value in `platforms/debian_sid.yml`)
    - *amd64* folder (`vm.metadata.target_namevalue )
      - tasks or vars files, ... specific for *amd64* targets in *debian_11* platforms
    - *arm64* folder (`vm.metadata.target_namevalue )
      - tasks or vars files, ... specific for *arm64* targets in *debian_11* platforms
- *fedora_36* folder (`vm.metadata.platform_name` value )
    - *amd64* folder (`vm.metadata.target_namevalue )
      - tasks or vars files, ... specific for *amd64* targets in *fedora_36* platforms
    - tasks or vars files, ... specific *fedora_36* platforms but any target
- tasks or vars files, ... generic for any platform and target which file does not exists with a specific `import_path` sub path

The `import_path` is useful when some dependencies have different alias in some platform's packets manager, or user needs "ad hoc" tasks/vars for some others use cases.

Requirements
------------

- ansible collections:
  - [community.libvirt](https://galaxy.ansible.com/community/libvirt) 
    - ```ansible-galaxy collection install community.libvirt```
- Packages
  - `python` >= 2.6
  - `python3-libvirt` ( community.libvirt dep )
  - `python3-lxml` ( community.libvirt dep )
  - `zipinfo` (unarchive module dep)
  - `zstd` to expand .tar.zst files (unarchive module optional dep)
  - `unzip` to handle .zip files (unarchive module optional dep)
  - `gtar` to handle .tar.* files (unarchive module optional dep)
  - `gzip` to handle .gz files (optional)
    - required **if** using unsupported archive format by the unarchive module
  - `bunzip2` to handle .bz2 files (optional)
    - required **if** using unsupported archive format by the unarchive module
  - `sshpass`
    - optional but required to use password on ssh on vm connections
    - otherwise use [ansible vault](https://docs.ansible.com/ansible/2.8/user_guide/vault.html)
  - `libvirt-bin`
    - required by `guest_provision` role to handle snapshots using virsh
  - Any `qemu-system-<architecture>` (if you are using qemu)
    - these must be installed on the hypervisor host before processing the VM installation
- System running hypervisor:
  - Supported platform:
    - Theoretically any GNU/Linux distribution
    - Tested:
      - debian
  - hypervisor
    - default: `qemu`
    - tested:
      - `qemu`
  - libvirt environment
    - libvirt daemon active and running
- Ansible User:
  - group: `libvirt_group` var value (default: `libvirt`)
    - require to have access to libvirt features
      - See your distribution requirements to use libvirt features
  - group: `kvm`
    - required to use KVM
  - group: `hypervisor_group` var value (default: `libvirt-qemu` )
    - required to change files ownership to allow the hypervisor to access it
      - in general see your hypervisor requirements
  - Note: default `hypervisor_group` and `libvirt_group` vars are defined in `roles/kvm_provision/defaults/main.yaml`, so they can be overridden on hypervisor's host (group)vars according to your use case.

For more details see each role's requirements

All prerequisites are satisfied by the usage help above, but if you need advanced usage these informations may help:

- `roles/guest_provision` requires that
  - each **already defined** VM host have
    - an already defined ansible_connection plugin with relative variables
    - The ansible controller can reach the VM and hypervisor
      - So there should be a network that connect the hypervisor to the VM
    - the `vm` var object with `VM definition` format (mainly required for `vm.metadata.name` and `vm.metadata.connection` uri)
    - the `kvm_host` var string with the value of the VM's hypervisor
  - Note: all these, expect for `ansible_connection`, are satisfied by using the `roles/init_vm_connection` utility
- `roles/kvm_provision` requires that
  - the `vm` var object as `VM definition` format

Dependencies
------------

- [community.libvirt](https://galaxy.ansible.com/community/libvirt)