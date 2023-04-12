init_vm_connection
=========

Ensure to add the VM definition as ansible inventory host, such that:

- Its inventory hostname is `"{{ vm.metadata.name }}"`
- it's a member of the following groups:
  - `vms`
  - `"{{ vm.metadata.name }}"`
  - `"{{ vm.metadata.platform_name }}"`
  - `"{{ vm.metadata.target_name }}"`

This role setup the Proxy Jumps inside an ssh configuration file for each nested `kvm_host` that are required by the VM to be reachable from the ansible controller when `ssh` or `paramiko_ssh` connection plugins are used to connect the VM.

This role defines also the libvirt network used by the `VM definition` if doesn't exist and add a DHCP entry using:
- mac address: `{{ vm.net.mac }}`
- ip: `{{ vm.net.ip }}`
- hostname: `{{ vm.metadata.hostname }}`

Requirements
------------
None

Role Variables
--------------
- `vm`
  - required
  - It's the `VM definition` object

The following vars are assigned to the VM's inventory such that:

- `vm` is the `VM definition`
- `kvm_host` is the `{{ inventory_hostname }}`of hypervisor hostname
- `ansible_hostname` is`{{ vm.metadata.hostname }}`
- `ansible_host`: this value depends by the connection method used by the VM and this value will be used as host name for the entries of the ssh configuration file.

  - `{{ vm.metadata.name }}` if using `community.libvirt.libvirt_qemu` as VM ansible connection
  - `{{ vm.metadata.hostname }}` if using `ssh` as VM ansible connection and `vm.net.type` is `user`
  - `{{ vm.net.ip }}` otherwise

- `ansible_real_host` is `{{ vm.net.ip | default( vm.metadata.hostname ) }}` used always as real address/hostname variable

  - Note: This has been added becase the `ansible_host` will be overrided by the `vm.metadata.hostname` when using the `user` network type and `ssh` plugin.
  This behavior is required since ansible ssh plugin search inside the ssh configuration file for an hostname alias entry matching with the `ansible_host` variable before connecting to the target node.
  So the implementation choice made consists on considering `ansible_host` as the host alias of each entry inside the the ssh configuration file, so `ansible_host` may be a real hostname or a fake one used as alias (usually used for nested VM).

- `hostjumps` is the list of inventory_hostname(s) required to reach the VM. Every VM keep this list updated when using this role by setting it as the `kvm_host`'s `hostjumps` + the `kvm_host` node name


Dependencies
------------

- `libvirt_network` role
  - used to define networks and add DHCP entries
- [ansible.utils](https://galaxy.ansible.com/ansible/utils)

Example Playbook
----------------

```
- name: Init VM on KVM
  hosts: hypervisors
  gather_facts: yes
  pre_tasks:
  - name: Loading vm definitions from file
    include_vars:
      file: vms_config.yaml
      name: my_vm_config
    # allow to override using -e
    when: not( vm_config is defined ) 
  
  roles:
  - role: parse_vms_definitions
    vars:
      config: "{{ my_vm_config }}"
  
  tasks:  
  # add VMs definitions to the ansible inventory
  - name: init VM connection
    loop: "{{ virtual_machines }}"

    include_role:
      name: init_vm_connection
    
    loop_control:
      loop_var: vm

```

License
-------

GPL-3.0-or-later

Author Information
------------------

None
