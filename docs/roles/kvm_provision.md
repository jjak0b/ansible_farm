kvm_provision
=========

This role configures and creates VMs on a hypervisor which supports libvirt API (QEMU, KVM, etc ...).

- Create: Allow to specify how a VM should be defined and installed into libvirt by using a `VM definition` and a `libvirt XML template`.

- Configure: Allow to specificy how a resource should be elaborated (an image disk, kernel, etc ...) by specifing `callback-tasks` .

## Vocabulary

- `platform`: synonym of OS; it's the definition of used OS; it's the definition used in each vm configuration to setup network, admin, user, vm images and other assets.
- `target`: synonym of machine, architecture of machine; it's the definition of the architecture type used in each vm configuration, like CPU, machine type, emulator, etc ...
- `VM definition` indicates the object which is a combination of nested target and platform definitions' properties used to create VM template in XML libvirt format.

## The VM definition
You can instanciate each [ VM definition ](objects/vm_definition.md) by using the `roles/parse_vms_definitions` utility role to create definitions from separated files organized by [ platforms ](objects/vm_definition.md#Platform_definition) and [targets](objects/vm_definition.md#Target_definition). See its documentations for more details.

A `VM definition` object is composed by 2 parts:

- metadata variables
- template dependent variables

The **metadata** variables are used to specify different info about the VM which any role of this collection can handle by using a documented API for different purposes:

- VM identification
- libvirt URI connection type
- Authentication info
- Resource processing and installation
- XML libvirt Template specification

The **template dependent** variables are used to specify custom variables to use for different purposes of the user but are mainly used to define the components that the virtual machine is made of. 

- **Why:** The Libvirt XML format is complex and a single role cannot handle all possible use cases, so the `kvm_provision` role provides you a default template that uses default template dependent variables for a simple use case, but you can specify your own template and use any custom variables you want for your use case.

**Note: the use of `parse_vms_definitions` role is recommended because both target and platform definitions use some default metadata properties, but some of these may me overrided according to your use case inside you VM definition.**

### Definition use details for this role

-  **Properties defined in `vm.metadata` are required and needs their respective API names because are also used internally by the roles of its collection**
- properties of `vm.metada.auth` may be declared instead as host/group variables adding the "ansible_" prefix to the property' names (for example ansible_user) and they depends by the connection method used to allow ansible to connect and login to the virtualized hosts.
- `vm.metadata.sources` are a list of resources that specify the workflow of a resource using `callback-tasks`.
For each item of the list: all `callback-tasks` defined into each entry of the `vm.metadata.sources[i].before_provision` list are processed first and then the `callback-task` specified by the `vm.metadata.sources[i].on_provision` property will be processed later.
  - A `callback-task` object has the following scheme:
    ```
    callback: The path of the callback task file to use, relative to the `tasks` directory
    ...: any others specific arguments used by the callback-task
    ```
    When defining a `callback-task` you can assume that in its task's scope can access to the following facts:
      - `source` and `source_index`: mapped to `vm.metadata.sources` 's items on declared order
      - `task` and `task_index`: mapped to each item of `vm.metadata.sources[i].before_provision` or the `vm.metadata.sources[i].on_provision` entry.
      - `vm`: the VM definition
    
  - **Why:** This role provide a **customizable and documented way** to: 
      - define the elaboration hooks of a VM resource through the `.before_provision` list of `callback-task` 
        - So you can pre-process an image by resizing it, adding users, change credentials, add ssh keys, etc ...
      - define the installation hook of a VM resource through the `.on_provision` entry
        - So you can install into libvirt using different ways you want, for example using different storage managment

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
        - `xz` command on `.xz` files
        - [ansible.builtin.unarchive](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/unarchive_module.html) module otherwise.
      - arguments:
        - `src`: The input archive file to extract
        - `dest`: The destination path to extract to
    - `callbacks/sources/copy.yaml`
      - Copy a remote resource to a destination relative to the `vm.metadata.libvirt_pool_dir` path on remote
      - arguments:
        - `src`: The relative resource path of the file to be copied
        - `dest`: The relative resource path destination
    - `callbacks/sources/img_convert.yaml`
      - Convert an image from a format to an other format using `qemu-img`
      - arguments:
        - `src`: The relative resource path of the file to convert
        - `dest`: The relative resource path destination of file
        - `from_format`: the image format of `src`
        - `to_format`: the image format of `dest`
    - `callbacks/sources/setup_image.yaml`
      - Setup an image using `virt-sysprep` to setup root password, hostname, SSH public key and first boot script that add user and SSH authorized key for the user. The credentials used for root and user are the same of the one set in `.metadata.auth` variables.
      - arguments:
        - `src`: The relative resource path of the file to be copied
        - `dest`: The relative resource path destination
    - `callbacks/sources/install_to_libvirt.yaml`
      - Move the resource to the `vm.metadata.libvirt_pool_dir` path
      - arguments:
        - `src`: The resource path to the file to be installed
        - `dest`: The final resource name to use into the installation directory
  - Other callbacks may be stored into the  _tasks_ or `hypervisor_lookup_dir_path` directories
    - You can override the built-in callbacks as you want
  - The `src` and `dest` parameters of the builtin callbacks should be relative to the `vm.metadata.tmp_dir` when they are related to a file path and you should use this convention for processing tasks.
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

### Default template dependant variables

The common and default definitions are provided in `defaults/targets/` and `defaults/platforms/` role folders and they are used by the default provided template.


Requirements
------------

- hypervisor target requirements

  - A libvirt environment already setup
  - Packages and commands
    - `python` >= 2.6
    - `python3-libvirt` ( community.libvirt dep )
    - `python3-lxml` ( community.libvirt dep )
    - `virsh`
    - Any emulator you will use in the VM XML template
      - any `qemu-system-<architecture>` emulator by default
    - `zstd` to expand .tar.zst files (unarchive module optional dep)
      - required **if** using `unarchive` callback-task with a supported `unarchive` module format
    - `unzip` for `zipinfo` command and to handle .zip and .tar.* files (unarchive module optional dep)
      - or `gtar` (unarchive module optional dep)
      - required **if** using `unarchive` callback-task with a supported `unarchive` module format
    - `gzip` to handle .gz files (optional)
      - required **if** using `unarchive` callback-task with an unsupported archive format by the unarchive module
    - `bzip2` to handle .bz2 files (optional)
      - required **if** using `unarchive` callback-task with an unsupported archive format by the unarchive module
    - `xz` to handle .xz files (optional)
      - required **if** using `unarchive` callback-task with an unsupported archive format by the unarchive module
    - `virt-sysprep`
      - required if using the `setup_image` callback-task
    - `qemu-img`
      - required if using the `img_convert` callback-task
  - platforms:
    - Supported: 
      - Any GNU/Linux distribution (in theory)
      - POSIX-compilant OS should work
    - Tested platforms:
        - Debian 11, 12
        - ArchLinux
  - Supported hypervisors:
    - Any hypervisor driver compatible with libvirt should work with this role
    - tested:
      - `kvm`
      - `qemu`
  - Ansible User:
    - group: `libvirt_group` var value (default: `libvirt`)
      - required **if** you need to use some libvirt features like libvirt network and `/system` URI
        - See your distribution requirements to use libvirt features
    - group: `kvm`
      - required **if** you use KVM
    - group: `hypervisor_group` var value (default: `libvirt-qemu` )
      - required  **if** you use `install_to_libvirt` callback-task to change files ownership to allow the hypervisor to access it
        - This is required if you specify an installation directory that the user don't own like `/var/lib/libvirt/images`
        - Your user should have the permission to do that, this role won't use `become: yes` because focus to be used by standard user first, so i recommend to create a custom installation callback-task to use your ansible_become_user for this type of operation
        - in general see your hypervisor requirements
    - Note: default `hypervisor_group` and `libvirt_group` vars are defined in `roles/kvm_provision/defaults/main.yaml`, so they can be overridden on hypervisor's host (group)vars according to your use case.

- ansible collections:
  - [community.libvirt](https://galaxy.ansible.com/community/libvirt) 
  - [community.crypto](https://galaxy.ansible.com/community/crypto) 
    - required **if** using `setup_image` callback-task
- ansible modules:
  - [unarchive](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/unarchive_module.html) 
    - required **if** using `unarchive` callback-task


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
  - optional (default: `setup_vm`, see defaults/main )
  - It's the lookup path for searching VM templates
- `hypervisor_lookup_dir_path`
  - optional (default: `setup_hypervisor`, see defaults/main )
  - It's the lookup path for searching `callback-tasks`
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
