kvm_provision
=========

This role configures and creates VMs on a hypervisor which supports libvirt API (QEMU, KVM, etc ...)

## Vocabulary

- platform: synonym of OS; it's the definition of used OS; it's the definition used in each vm configuration to setup network, admin, user, vm images and other assets.
- target: synonym of machine, architecture of machine; it's the definition of the architecture type used in each vm configuration, like CPU, machine type, emulator, etc ...
- `VM definition` indicates the object which is a combination of nested target and platform definitions' properties used to create VM template in XML libvirt format.
## Target definition
A target definition has the following scheme example:
```
vm:
  metadata:
    # This is the architecture name alias given to this target, it's the target identifier.
    # It's used to reference this when we use the term 'target name', (e.g. for `import_path` into guest lifecycle's tasks)
    arch_name: "amd64"

  virt_domain: "qemu" or "kvm" 

  # architeture name alias used by the VM XML definition
  arch: "x86_64"  

  # emulator path of this target into the KVM host, if virt_domain = "kvm" may be '/usr/bin/kvm' 
  emulator: "/usr/bin/qemu-system-x86_64" 

  # cpu name to emulate on this target, set this only if virt_domain is not 'kvm'
  cpu: qemu64

  # Machine type used by the emulator
  machine: "q35" 
```
## Platform definition 
A platform definition has the following scheme example:
```
vm:
  # metadata properties are used inside a task for installation and setup purposes
  metadata:
    template: "mytemplateName" # will use the default.xml.j2 if this is not set
    # VM name identifier 
    name: &vm_name "VM {{ arch_name }}"
    # VM guest hostname
    hostname: *vm_name
    connection: "qemu:///system" # "qemu:///session"  
    # directory to store VM images
    libvirt_pool_dir: "/var/lib/libvirt/images/" # libvirt_pool_dir: "{{ansible_env.HOME}}/.local/share/libvirt/images/"

    # auth credentials to auth as user or become_user (root/admin) into this platform
    auth:
      become_user: "root / admin user"
      become_password: "root / admin password"
      become_method: "su" # see https://docs.ansible.com/ansible/latest/user_guide/become.html 
      user: "user name"
      password: "user password"
    # assets to be downloaded and processed, required for VM install 
    sources:
    # any uri to any file 
    - uri: "myURI/myImage-{{ arch_name }}.qcow2.tar.gz"
      checksum_uri: "myURI/myImage.qcow2.tar.gz.sha1sum"
      checksum_type: "sha1"
      # fallback checksum value
      checksum_value: "onlyThisIsSupported"
      # image destination name after unarchived / processed, and moved to 'libvirt_pool_dir'
      # for unarchive process, are supported: .gz, .bz2 and https://docs.ansible.com/ansible/latest/collections/ansible/builtin/unarchive_module.html
      unarchived: &image_file_name "myImage.qcow2"
    cleanup_tmp: no # if truthy will delete downloaded sources which are not into installation directory

  # Here can be inserted others fully custom properties for templating a custom vm's xml

  # Some properties here, if needed, could override specific architecture values for all architectures
  vcpus: 4 # host cpu cores assigned to VM
  ram: 2048 # MB 
  net:
    type: 'user' or 'network' or 'bridge'
    source: 'default' | 'a network name' | 'virbr0'
    mac: "52:54:00:ca:cc:ac"
    # ipv4 for now is supported 
    ip: "192.168.100.1" or 'dhcp'
    mask: "24" # only if ip is not 'dhcp' 
  disks:
  - type: "raw"
    devname: "hda"
    src: *image_file_name
```

You can instanciate the `VM definitions` by using the `roles/parse_vms_definitions` utility role to create definitions from separated files organized by platform and targets. See its documentations for more details.
**Note: the use of `parse_vms_definitions` role is recommended because both target and platform definitions use a default definition, but some of the properties must me overrided according to your use case.**

### Definition use details
- properties of `vm.metada.auth` may be declared instead as host/group variables adding the "ansible_" prefix to the property' names (for example ansible_user) and they depends by the connection method used to allow ansible to connect and login to the virtualized hosts.
- Each uri entry of `vm.metadata.sources` :
  - is fetched using [ansible.builtin.get_url](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/get_url_module.html) module which supports HTTP, HTTPS, or FTP URLs
  - is extracted if the `.unarchived` 's filename is different than the `.uri`'s filename
-  **Properties defined in `vm.metadata` needs their respective API names because are also used internally by the roles**
-  Other properties in the `vm` root, except for `vm.metadata`, can be customized or renamed if you are using a custom XML template for VM definitions.
   - For instance `vcpus` is a property that is used by the default template but the variable name can be changed if you are using a custom XML template and use the new reference name inside of it.
   - About the custom variables the important thing is that they must be defined in final `VM definition` but **it doesn't matter where the custom variable is defined** since both definitions are merged together and so they can be defined into the target or platform definitions.
### Run platform dependent tasks
On `roles/parse_vms_definitions` processing you can run custom tasks defined in a yaml file placed in `tasks/platforms/<platform_name>.yml` before the merge of target and platform vars and before the template is processed.
A use case for this can be an utility to override the internal var `platform_vars.vm` / `target_vars.vm` according to your use case
For instance:
```
# run some checks and we require to change resources' uri and other stuff
# ensure to compile stuff for specific target_vars.vm.metadata.arch_name, and serve it locally and override the uris in vm.metadata.sources if needed
# ... 
- name: Override platform vars for some specific use case
  set_fact: 
    platform_vars: "{{ platform_vars | combine (override_platform_vars, recursive=True) }}"
  vars:
    override_platform_vars: 
      vm:
        vcpu: 4
        # ... and any other field 
  
```

## VM template XML definition
A `default.xml.j2` VM template definition scheme has been provided in `roles/kvm_provision/templates`
Otherwise custom templates are applied, searching into `templates` folder with the matching name and the following priority order:
- `{{ vm.metadata.template }}.xml.j2`
- `{{ vm.metadata.name }}.xml.j2`
- `{{ vm.arch }}.xml.j2`

Each template during the templating process can access to the `vm` root property of the `VM configuration` to define a dynamic XML for each VM. Then a custom template can be fully customizable.

For XML advanced use, see [https://libvirt.org/format.html](https://libvirt.org/format.html).

Requirements
------------

- hypervisor Requirements
  - Before processing the VM installation, some system components may be required according to the vm `target` type.
  This role will run the tasks on the KVM/Hypervisor host specified in:
    - `{{ hypervisor_lookup_dir_path }}/prerequisite/targets/{{ vm.arch }}.yml` if exists
    - `defaults/prerequisite/targets/{{ vm.arch }}.yml` otherwise.
  
    Some qemu target architectures are already defined (see folder in `roles/kvm_provision/defaults`) but according to your use case you can create a custom `prerequisite/{{ vm.arch }}.yml` file to override them or create a new architecture prerequisite's tasks file.
    - For example: 
      If your `VM definition` object defines `vm.arch: aarch64`, then you can create a `{{ hypervisor_lookup_dir_path }}/prerequisite/targets/aarch64.yml` containing
      ```
      - name: Ensure arch System components
        package:
          name:
            - qemu-system-arm
          state: present
        become: yes
      ```
- ansible collections:
  - [community.libvirt](https://galaxy.ansible.com/community/libvirt) 
    - ```ansible-galaxy collection install community.libvirt```
- ansible modules:
  - [unarchive](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/unarchive_module.html) 
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
- `virtual_machines`
  - required
  - it's a list of `VM definition`
- `parse_lookup_dir_path`
  - optional (default: see defaults/ )
  - It's the lookup path for searching VM templates
- `hypervisor_lookup_dir_path`
  - optional (default: see defaults/ )
  - It's the lookup path for searching hypervisor prerequisite tasks
- `libvirt_group`: 
  - optional (default: libvirt)
- `hypervisor_group`:
  - optional (default: libvirt-qemu)

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
            arch_name: "armvirt64"
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
            - uri: "myURI/myImage-armvirt64.qcow2.tar.gz"
              checksum_uri: "myURI/myImage-armvirt64.qcow2.tar.gz.sha1sum"
              checksum_type: "sha1"
              checksum_value: "onlyThisIsSupportedForNow"
              unarchived: &image_file_name "myImage.qcow2"
            cleanup_tmp: no
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

BSD

Author Information
------------------

None
