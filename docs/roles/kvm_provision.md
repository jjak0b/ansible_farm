kvm_provision
=========

This role configures and creates VMs on a hypervisor which supports libvirt API (QEMU, KVM, etc ...)

## Vocabulary

- `platform`: synonym of OS; it's the definition of used OS; it's the definition used in each vm configuration to setup network, admin, user, vm images and other assets.
- `target`: synonym of machine, architecture of machine; it's the definition of the architecture type used in each vm configuration, like CPU, machine type, emulator, etc ...
- `VM definition` indicates the object which is a combination of nested target and platform definitions' properties used to create VM template in XML libvirt format.

## The VM definition
You can instanciate each [ VM definition ](objects/vm_definition.md) by using the `roles/parse_vms_definitions` utility role to create definitions from separated files organized by [ platforms ](objects/vm_definition.md#Platform_definition)) and [targets](objects/vm_definition.md#Target_definition). See its documentations for more details.

**Note: the use of `parse_vms_definitions` role is recommended because both target and platform definitions use a default definition, but some of the properties must me overrided according to your use case.**

### Definition use details
-  **Properties defined in `vm.metadata` are required and needs their respective API names because are also used internally by the roles of its collection**
- properties of `vm.metada.auth` may be declared instead as host/group variables adding the "ansible_" prefix to the property' names (for example ansible_user) and they depends by the connection method used to allow ansible to connect and login to the virtualized hosts.
- Each item of `vm.metadata.sources` is first processed by all `callback-tasks` defined into each entry of the `vm.metadata.sources[i].before_provision` list and after processed by the `callback-tasks` specified by the `vm.metadata.sources[i].on_provision` property.
  - A `callback-task` has the following scheme:
    ```
    callback: The path of the callback task file to use, relative to the `tasks` directory
    ... others specific arguments used by the callback-task
    ```
    When defining a `callback-task` you can assume that in its task's scope can access to the following facts:
      - `source` and `source_index`: mapped to `vm.metadata.sources` 's items on declared order
      - `task` and `task_index`: mapped to each item of `vm.metadata.sources[i].before_provision` or the `vm.metadata.sources[i].on_provision` entry.
      - `vm`: the VM definition
  - The `before_provision` entry lists any pre-processing `callback-tasks` that the i-th (re)source needs before being installed.
  - The `on_provision` entry define the last `callback-task` that the i-th (re)source needs to be installed. This type of callback will be processed after all sources have processed their `before_provision`'s `callback-tasks`.
  If the `.callback` property is not provided then the `callbacks/sources/install_to_libvirt.yaml` callback is going to be used implicitly.
  - Some built-in provided `callback-task` are:
    - `callbacks/sources/fetch.yaml`
      - use [ansible.builtin.get_url](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/get_url_module.html) module which supports HTTP, HTTPS, or FTP URLs.
      it will use the cached resource if available, otherwise will fetch the resource.
      - arguments:
        - `src`: The URI of the resource to fetch 
        - `dest`: The resource name of the URI to be renamed to
    - `callbacks/sources/unarchive.yml`
      - extract the `src` file to the `dest` directory by using:
        - `gzip` command on `.gz` or `.tar.gz` files
        - `bunzip2` command on `.bz2` files
        - [ansible.builtin.unarchive](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/unarchive_module.html) module otherwise.
      - arguments:
        - `src`: The input archive file to extract
        - `dest`: The destination path to extract to
    - `callbacks/sources/install_to_libvirt.yaml`
      - Move the resource to the `vm.metadata.libvirt_pool_dir` path
      - arcguments:
        - `src`: The resource path to the file to be installed
        - `dest`: The final resource name to use into the installation directory
  - Other callbacks may be stored into the  _tasks_ or `hypervisor_lookup_dir_path` directories
    - You can override the built-in callbacks as you want
  - The `src` and `dest` of the builtin callbacks when are relative to the `vm.metadata.tmp_dir` when they are related to a file path. 
-  Other properties in the `vm` root, except for `vm.metadata`, can be customized or renamed if you are using a custom XML template for VM definitions.
   - For instance `vcpus` is a property that is used by the default template but the variable name can be changed if you are using a custom XML template and use the new reference name inside of it.
   - About the custom variables the important thing is that they must be defined in final `VM definition` but **it doesn't matter where the custom variable is defined** since both definitions are merged together and so they can be defined into the target or platform definitions.

## VM template XML definition
A `default.xml.j2` VM template definition scheme has been provided in `roles/kvm_provision/templates`
Otherwise custom templates are applied, searching into `templates` folder with the matching name and the following priority order:
- `{{ vm.metadata.template }}.xml.j2`
- `{{ vm.metadata.name }}.xml.j2`
- `{{ vm.arch }}.xml.j2`

Each template during the templating process can access to the `vm` root property of the `VM definition` to define a dynamic XML for each VM. Then a custom template can be fully customizable.

For advanced libvirt XML, see its [format documentation](https://libvirt.org/format.html).

### Default template's variable dependencies

The common and default definitions provided in `defaults/targets/` and `defaults/platforms/` role folders and they are used by the default provided template: All `vm`'s properties defined outside the `vm.metadata` are considered as "template dependencies".


Requirements
------------

- hypervisor Requirements
  - System components like `qemu-system-<architecture>` if you are using qemu
    - these must be present before processing the VM installation
- ansible collections:
  - [community.libvirt](https://galaxy.ansible.com/community/libvirt) 
    - ```ansible-galaxy collection install community.libvirt```
  - [community.crypto](https://galaxy.ansible.com/community/crypto) 
    - ```ansible-galaxy collection install community.crypto```
- ansible modules:
  - [unarchive](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/unarchive_module.html) 
- Packages
  - `python` >= 2.6
  - `python3-libvirt` ( community.libvirt dep )
  - `python3-lxml` ( community.libvirt dep )
  - `zstd` to expand .tar.zst files (unarchive module optional dep)
  - `unzip` for `zipinfo` command and to handle .zip and .tar.* files (unarchive module optional dep)
    - or `gtar` (unarchive module optional dep)
  - `gzip` to handle .gz files (optional)
    - required **if** using unsupported archive format by the unarchive module
  - `bzip2` to handle .bz2 files (optional)
    - required **if** using unsupported archive format by the unarchive module
  - `libvirt-clients`
    - required to use the `virsh` command
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

Role Variables
--------------
- `vm`
  - required
  - it's a `VM definition` object
- `create_vm`: (default: true)
  - if true wil create VM if not exists, otherwise won't install any VM
- `delete_vm`: (default: false)
  - if true will stop and delete VM (without resource cleanup)
- `should_remove_all_vm_storage`: (default: false)
  - if true will add the flag --remove-all-storage to remove all storage assets when `delete_vm` is true, none otherwise
- `parse_lookup_dir_path`
  - optional (default: see defaults/main )
  - It's the lookup path for searching VM templates
- `hypervisor_lookup_dir_path`
  - optional (default: see defaults/main )
  - It's the lookup path for searching hypervisor prerequisite tasks
- `libvirt_group`: 
  - optional (default: *libvirt*)
- `hypervisor_group`:
  - optional (default: *libvirt-qemu*)

template scope:

- `vm` definition object is accessibile into XML.j2 templates 

Dependencies
------------

- `roles/parse_vms_provision`
  - optional utility to provide its output in - `virtual_machines` fact
  

Example Playbook
----------------
```
- hosts: hypervisors
  tasks:
  - name: create virtual machines definitions from files
    include_role: parse_vms_definitions
    vars:
      config: "{{ define }}" # see its role doc 
  - name: Manual VM def
    set_fact:
      some_other_vms:
      - vm:
          metadata:
            target_name: "armvirt64"
            name: "my VM name for armvirt64"
            hostname: "debian-armvirt64"
            connection: "qemu:///system"
            libvirt_pool_dir: "{{ansible_env.HOME}}/.local/share/libvirt/images/"
            auth:
              become_user: "root"
              become_password: "root"
              become_method: "su" 
              user: "root"
              password: "user"
            sources:
            - before_provision:
              - callback: "callbacks/sources/fetch.yaml"
                src: "myURI/myImage-armvirt64.qcow2.tar.gz"
                dest: "myImage-armvirt64.qcow2.tar.gz"
              - callback: "callbacks/sources/unarchive.yml"
                src: "myImage-armvirt64.qcow2.tar.gz"
                dest: "./"
              - callback: "myCustomProcessingTask.yaml"
                src: "myImage-armvirt64.qcow2"
                parameterA: ...
                parameterB: ...
              on_provision:
                src: myImage-armvirt64.qcow2
                dest: "version-X_myImage-armvirt64.qcow2"  
            template: "virtio"
          arch: "aarch64"
          net:
            type: 'network'
            source: 'default'
            mac: "52:54:00:ca:cc:ac"
            ip: "192.168.100.1"
            mask: "24"
          # all vars that depends by the used template ( in this example would be "virtio.xml.j2")
          emulator: "/usr/bin/qemu-system-aarch64" 
          virt_domain: "kvm" 
          machine: "virt" 
          vcpus: 4
          ram: 2048
          disks:
          - type: "raw"
            devname: "hda"
            src: *image_file_name
      - ...
  - name: KVM Provision role
    include_role: kvm_provision
    vars:
      virtual_machines: "{{ virtual_machines | union( some_other_vms ) }}
```

License
-------

GPL-3.0-or-later

Author Information
------------------

None
