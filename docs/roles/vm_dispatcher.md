vm_dispatcher
=========

Fetch hypervisor capabilities like supported emulator per architectures, and distribute `VM configurations` over hypervisors that are playing this role.
A VM would be assigned to a compatible hypervisor that support its architecture.
the assignment takes place with the following criteria:
- The variable `priority_domains` specify which virtualization type should have the priority (kvm, qemu, etc ...)
  - The first domain type which a hypervisor has support with for a specific architecture will be chosen as candidate for the assignment.
- If multiple hypervisors supports the same architecture, then the assignment will take place with a fair queue


Requirements
------------

- [virsh](https://www.libvirt.org/manpages/virsh.html) command
- libvirt environment
- Assign the `VM configuration` you need to assign in `config` variable on `localhost` inventory hostname.

Role Variables
--------------

This role use the following variable parameters:

- `config`: The `VM configuration` **set on** `localhost` used to parse the VM targets and assign to hypervisor.
  - **Note**: This is very important since the `localhost` node is used as unique and intermediary node to store the `VM configuration` that shouldn't overlap with the `VM configurations` statically assigned on the hypervisors nodes.
- `ansible_libvirt_uri`: (optional) The libvirt URI used by `virsh` to fetch the libvirt capabilities
  - if provided it may provide more infos, otherwise `virsh` will use the default one (see `virsh` docs for this)
- `priority_domains`: ( default: `[ kvm, qemu ]` )  List of virtualization domain type supported by libvirt
This role set the following useful public variables:

- `libvirt_supported_virtualizzation_types`: dictionary of list of domains/virtualization types grouped by libvirt architecture alias
- `config`: On each target assign a VM configuration based on the assignment.
  - **Note** that this will not override the existing `VM configuration` set on the targets, but will append to it as `list`

Dependencies
------------

- `parse_vms_definition`: Used to parse the VM's target to check the `vm.arch` it require to emulate.

Example Playbook
----------------

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

    - hosts: localhost
      connection: local
      tasks:
        - name: This may be set also in localhost inventory
          set_fact:
            config: 
              permutations:
                targets:
                  - amd64
                  - arm64
                platforms:
                  - debian
                  - fedora
              list:
                - target: amd64
                  platform: 'Arch-Linux'

    - hosts: hypervisors
      roles:
         - { 
              role: vm_dispatcher,
              ansible_libvirt_uri: qemu:///session,
           }

License
-------

GPL-3.0-or-later

Author Information
------------------

None
