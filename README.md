# TEST FARM

An ansible tool to provision an host with VMs of different machine, architecture and OS configs and to provision VMs with custom tasks
## Usage

If you want use custom ansible config, you can use `ANSIBLE_CONFIG="./ansible.cfg"` first.
### KVM host provisioning: Install and setup VMs on KVM hosts
```
ansible-playbook main.yaml -K
```
By default the VMs configurations are created from the definitions provided into `vars/vm_definitions.yaml`.
### KVM guest provisioning: Install dependencies on VM guests and run the guest lifecycle phases's
```
ansible-playbook guest_provision.yaml
```
By default the VMs info use the configurations parsed using `roles/parse_vms_definitions` and from thee definitions provided into `vars/vm_definitions.yaml`.

## Vocabulary

- platform: synonym of OS; it's the configuration of used OS; it's the configuration used in each vm configuration to setup network, admin, user, vm images and other assets.
- target: synonym of machine, architecture of machine; it's the configuration of the architecture used in each vm configuration, like CPU, machine type.

## VMs definitions
By default the VMs configurations are created from the definitions provided into `vars/vm_definitions.yaml`.
But for advanced customization, the can be instanciated using `roles/parse_vms_definitions` from an vm_definitions object.
A vm_definitions object is a yaml, with the following scheme:
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
Each permutation and combination data create a lists of `VM definition` which are merged together.

A `VM definition` combine the target and platform definition data.
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
Properties defined in vm.metadata needs their respective name because are also used internally, but other properties can customized or renamed if using a custom XML template for VM definitions.
For instance `vcpus` can be defined instead into the target definition if using the default XML template (it doesn't matter where it's located but it's important that is defined in target or platform definition), or change the property name if using a custom one and use the changed name into the new xml into the right place.

**Note: both target and platform definitions use a default yaml, but some of the properties must me overrided according to your use case.**
## VM template XML definition
A `default.xml.j2` VM template definition scheme has been provided in `roles/kvm_provision/templates`
Otherwise custom templates are applied, searching into `templates` folder with the matching name and the following priority order:
- `{{ vm.metadata.template }}.xml.j2`
- `{{ vm.metadata.name }}.xml.j2`
- `{{ vm.arch }}.xml.j2`
A template name can be provided through a `template` field into a `VM definition` which will override the `vm.metadata.template` field.
Each template during the templating process can access to the `vm` root property of the `VM configuration` to define a dynamic XML for each VM. Then a custom template can be fully customizable.

For XML advanced use, see [https://libvirt.org/format.html](https://libvirt.org/format.html).

## The VM Guest lifecycle
The VM Guest lifecycle is decribed by `roles/guest_provision/tasks/guest_main.yml`

1. **dependencies** phase
   - Create 'clean' snapshot
   - Install use case dependencies
     - Import var dependencies if any (`vars/phases[/import_path]/dependencies.yaml`)
       - Install defined dependencies ( uses [ansible.builtin.package module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/package_module.html))
     - Run dependencies tasks (`tasks/phases[/import_path]/dependencies.yaml`)
   - Create 'dependencies' snapshot
2. **Init** use case phase: 
   - Run init tasks `tasks/phases[/import_path]/init.yaml`
   - Create 'init' snapshot
3. **Main** use case phase: 
   - Run main tasks `tasks/phases[/import_path]/main.yaml`
4. **Terminate** use case phase: 
   - Run end tasks `tasks/phases[/import_path]/terminate.yaml`

Where `import_path` is optional and it's a subpath of 2 nested folders if the use case needs specific tasks/vars for a target on platform or only platform; for instance:
- *debian_11* folder (`vm.metadata.platform_name` value in `platforms/debian_sid.yml`)
    - *amd64* folder (`vm.metadata.arch_name` value in `targets/amd64.yml`)
      - tasks or vars files, ... specific for *amd64* targets in *debian_11* platforms
    - *arm64* folder (`vm.metadata.arch_name` value in `targets/arm64.yml`)
      - tasks or vars files, ... specific for *arm64* targets in *debian_11* platforms
- *fedora_36* folder (`vm.metadata.platform_name` value in `platforms/fedora_36.yml`)
    - *amd64* folder (`vm.metadata.arch_name` value in `targets/amd64.yml`)
      - tasks or vars files, ... specific for *amd64* targets in *fedora_36* platforms
    - tasks or vars files, ... specific *fedora_36* platforms but any target
- tasks or vars files, ... generic for any platform and target which file does not exists with a specific `import_path` sub path

This is useful when some dependencies have different alias in some platform's packets manager, or user needs "ad hoc" tasks/vars for some others use cases.

