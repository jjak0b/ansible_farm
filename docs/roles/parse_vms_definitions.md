parse_vms_definitions
=========

Utility role which combine `targets` and `platforms` definitions files from an `VM configurations` object to instanciate a list of `VM definitions` to provide to the `kvm_provision` role.
The output list of is stored into `virtual_machines` fact.
The platforms vars overwrite the target nested vars when creating a `VM definition`.

Requirements
------------

The `targets` and `platforms` definitions files which this role use to create a `VM definition` for earch `VM configuration` have to be placed respectively into your `playbook dir/{{ parse_lookup_dir_path }}/platforms/` and `playbook dir/{{ parse_lookup_dir_path }}/targets/` dirs.
Each file definition file must be a YAML named as `your_platform_name` with `.yaml` or `.yml` extension.

The definition files are considered as YAML containing ansible tasks and **they must declare their definition by setting the fact named `vm`** as variable name of each platform / target definition object: For example your `<target name>.yaml` and/or `<platform name>.yaml` must contains at least the following task:
```
# ... some tasks user needs for some reason that needs for the specific platform / target
- name: defining my custom definition
  set_fact:
    vm:
      metadata:
        ...
    ...
```

This role includes the definition tasks in following order:

- target definition
- `defaults/platforms/defaults.yaml` if exists to use as shared initial `vm` fact
- platform definition

**Note**: Each `set_fact` on the a new `vm` fact will "override" its old `vm` fact value, but when the definition file returns, then the last previously defined `vm` fact will be merged internally with the new `vm` fact, such that each definition file can access to the previous `vm` fact value and add/overwrite the properties' values, but can't remove them.

While setting the `vm` fact, its sub-properties may depends by other variables:

- if they depends by some (global) variables then you can reference them directly using jinja expressions.
- if they depends by the target or platform 's names then you can access to the `vm.metadata.target_name` or `vm.metadata.platform_name` values or any sub-property of `vm` defined on previous definition file.


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
  - has the following properties:
    ```
    # There will be a VM of each platform for each target
    permutations: 
      # lists of target's yaml filename (no extension)
      targets: [] 
      # lists of platform's yaml filename (no extension)
      platforms: [] 

    # There will be this list of VMs with specific platform and target
    list: 
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
- hosts: hypervisors
  gather_facts: yes
  vars:
    # we 
    parse_lookup_dir_path: 'setup_vm'

    my_vm_config:
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
      list: 
        - platform: debian_11
          target: armvirt64
          template: virtio
  
  roles:
  - role: parse_vms_definitions
    vars:
      config: "{{ my_vm_config }}"
```
License
-------

BSD

Author Information
------------------

None
