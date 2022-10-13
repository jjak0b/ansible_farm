init_vm_connection
=========

Ensure to add the VM definition as ansible inventory host, suck that:
- Its inventory hostname is `"{{ vm.metadata.name }}"`
- it's a member of the following groups:
  - `vms`
  - `"{{ vm.metadata.name }}"`
  - `"{{ vm.metadata.platform_name }}"`
  - `"{{ vm.metadata.arch_name }}"`

The following vars are assigned to the VM's inventory such that:
- `ansible_host`: `{{ vm.net.ip }}` or `{{ vm.metadata.name }}` if using `community.libvirt.libvirt_qemu` as VM ansible connection
- `ansible_hostname`: `{{ vm.metadata.hostname }}`
- `vm` : the `VM definition`
- `kvm_host` the `{{ inventory_hostname }}`of hypervisor hostname

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

Dependencies
------------

- `libvirt_network` role
  - used to define networks and add DHCP entries

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

BSD

Author Information
------------------

None
