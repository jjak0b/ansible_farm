libvirt Network
=========

An utility role used to

- define a libvirt network by name and using associated template and configuration file
- add DHCP entries to a network

You can define a custom template and configuration file in `{{ net_parse_lookup_dir_path }}/networks/` otherwise `defaults/networks/` are used

Requirements
------------

- `community.libvirt.virt_net`

The .yaml configuration filename must be same of libvirt network name and must have this network configuration scheme using default template:
```
template: default
name: default-net
bridge: 
  name: virbr1
  stp: True
  mac: "52:54:00:00:00:01"
  ip: "10.0.0.254"
  netmask: "255.0.0.0"
dhcp:
  start: "10.0.0.1"
  end: "10.0.0.253"
```
Otherwise you can define a custom network configuration and template files if you want more customized networks type.
The network templates depends by the network configuration, so if you want to reuse a specific .xml.j2 template for your network configuration, specify it in the `template` property.

Role Variables
--------------

- `net_parse_lookup_dir_path`: lookup root directory used to search network templates and configuration
- `name`: network name of .yml (without extension) configuration file
- 

Dependencies
------------

None

Example Playbook
----------------
```

- hosts: vms
  vars:
    vm: ...
  tasks:
  - block:
    - name: Define Network
      include_role: 
        name: libvirt_network
        tasks_from: define
    - name: Insert DHCP entry
      include_role:
        name: libvirt_network
        tasks_from: add
      vars:
        mac: "{{ vm.net.mac }}"
        ip: "{{ vm.net.ip }}"
        hostname: "{{ vm.metadata.hostname }}"
    vars:
      name: "{{ vm.net.source }}"
      libvirt_uri: "{{ vm.metadata.connection | default(omit, true) }}"
    when: vm.net is defined and vm.net.type == 'network'
```

License
-------

BSD

Author Information
------------------

None
