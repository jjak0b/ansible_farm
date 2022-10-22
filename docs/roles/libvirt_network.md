libvirt Network
=========

An utility role used to

- define a libvirt network by name and using associated template and configuration file
- add DHCP entries to a network

You can define a custom template and configuration file in `{{ net_parse_lookup_dir_path }}/networks/` otherwise `defaults/networks/` are used

Requirements
------------

Note: The ansible user must have the right capabilities to be able to handle libvirt networking. See libvirt documentation for more details.

The .yaml configuration's filename must be same of libvirt network name.
if you are using the default network template you have to use the following scheme in your configuration:
```
template: the template name you are using (default)
name: name of the libvirt network (default net: default-net)
bridge: 
  name: name of the bridge associated to this network
  stp: boolean if you want stp enable on the bridge
  mac: mac address of the bridge
  ip: bridge's ip (IPv4)
  netmask: bridge's netmask
dhcp:
  start: Start ip of DHCP pool
  end: End ip of DHCP pool
```
Otherwise you can define a custom network configuration and template files if you want more customized networks type.
The network templates depends by the network configuration, so if you want to reuse a specific .xml.j2 template for your network configuration, specify it in the `template` property.

Role Variables
--------------

- `net_parse_lookup_dir_path`: lookup root directory used to search network templates and configuration
- `name`: network name of .yml (without extension) configuration file
- `libvirt_uri`: the libvirt uri

Dependencies
------------

- `community.libvirt`

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
