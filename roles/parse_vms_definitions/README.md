parse_vms_definitions
=========

Utility role which combine `targets` and `platforms` definitions files from an `VM configurations` object to instanciate a list of `VM definitions` to provide to the `kvm_provision` role.
The output list of is stored into `virtual_machines` fact.
The platforms vars ovveride the target nested vars when creating a `VM definition`.

Requirements
------------

The `targets` and `platforms` definitions files which this role use to create a `VM definition` for earch `VM configuration` have to be placed respectively into your `playbook dir/{{ parse_lookup_dir_path }}/platforms/` and `playbook dir/{{ parse_lookup_dir_path }}/targets/` dirs.
Each file definition file must be a YAML named as `your_platform_name` with `.yml` extension.
The definition file is considered as a vars file and must contains the `vm` variable as root var name

Role Variables
--------------
Note: See `defaults/main` default values for all optional vars
- `parse_lookup_dir_path` is the root search path used to search for matching target and platforms definition files
  - optional
  - For instance:
    - if `parse_lookup_dir_path` ha value `vm_setup`, then will search definitions in `vm_setup/targets/` and `vm_setup/platforms/` folders
    - otherwise if any of these is missing then will search definitions into `defaults/targets/` and `defaults/platforms/` folders of the role as fallback
- `config` it's a `configuration` object
  - required
  - has following properties:
    ```
    # There will be a VM of each platform for each target
    permutations: 
      # lists of target's yaml filename (no extension)
      targets: [] 
      # lists of platform's yaml filename (no extension)
      platforms: [] 

    # There will be this list of VMs with specific platform and target
    definitions: 
      - platform: platform's yaml filename (no extension)
        target: target's yaml filename (no extension)
        template: dedicated's xml.j2 filename (no extension)
      - ...
      ...
    ```
  - A template name can be provided through a `template` field into a `definitions` list item which will override the `vm.metadata.template` field.

Dependencies
------------

None

Example Playbook
----------------

```
- hosts: servers
  vars:
    # we 
    parse_lookup_dir_path: 'setup_vm'

    define:
      permutations: 
        targets:
        - amd64
        - bcm27XX
        - armvirt
        - armvirt64
      platforms:
        - debian_stable
        - debian_sid
        - fedora_stable
      definitions: 
        - platform: debian_11
          target: armvirt64
          template: virtio
  tasks:
  - name: create virtual machines definitions
    include_role: parse_vms_definitions
    vars:
      config: "{{ define }}"
```
License
-------

BSD

Author Information
------------------

None
