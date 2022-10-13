# VM definition

The `VM definition` object describes the characteristics of a VM about the hardware it emulates, the firmware or software that is installed onto it and other parameters like credentials and network host configuration.

## Scheme 
The scheme of a `VM definition` is the merge result of a `target` and a `platform` definition.

## Target definition
A target definition has the following require scheme:
```
vm:
  metadata:
    arch_name:  the architecture name alias given to this target, it's the target identifier.
```
Additional (target) properties that depends by the default template
```
vm:
  arch:         emulated architeture name alias used by the libvirt XML Domain format like "x86_64", "aarch64", etc ...
  emulator:     emulator path of this target into the Hypervisor host, for example "/usr/bin/qemu-system-x86_64" 
  virt_domain:  libvirt domain used like "qemu", "kvm", etc ...
  cpu:          cpu name to emulate this target like 'qemu64'. set this only if virt_domain is not 'kvm'
  machine:      Machine type that describe the VM like "q35", "armvirt", etc ...
```
## Platform definition 
A platform definition has the following required scheme

```
vm:
  # metadata properties are used inside a task for installation and setup purposes
  metadata:
    name: The VM name domain identifier 
    hostname: The VM guest hostname
    connection: The libvirt uri like "qemu:///system" or"qemu:///session"  
    libvirt_pool_dir: path to the directory where to store VM images
    # auth credentials to auth as user or become_user (root/admin) into this platform
    auth:
      user: "user name"
      password: "user password"
      become_user: The root / admin user
      become_password: The root / admin password
      # see https://docs.ansible.com/ansible/latest/user_guide/become.html 
      become_method: command used to perform the privilage escalation like "su" 
      
    # assets to be downloaded and processed, required for VM install 
    sources: List of resources
    - uri: Any URI relative to any file like images, archives, etc ... that need to be fetched
      checksum_type: Checksum algorithm alias name (like 'sha1') used for the checksum of the uri's resource
      checksum_uri: Any URI relative to any file that contains the value of the checksum associated to the uri's filename
      checksum_value: The checksum string value of the resource used as fallack if checksum_uri is not defined
      unarchived: The relative path to the working directory of the resource that should be installed into the 'libvirt_pool_dir'. If it's different than the uri's filename then it's considered as the extracted resource from the uri's file
    cleanup_tmp: boolean indicating if the uri's resource should be deleted

  net:
    type: libvirt network type: 'user' or 'network' or 'bridge' or 'vde'
    source: Network'default' or 'a network name' | 'virbr0'
    mac: "52:54:00:ca:cc:ac"
    # ipv4 for now is supported 
    ip: "192.168.100.1" or 'dhcp'
    mask: "24" # only if ip is not 'dhcp' 

  # Here can be inserted others fully custom properties for templating a custom vm's xml

```
Additional (platform) properties that depends by the default template
```
# if needed, these could override specific target definition properties
vm:
  vcpus: host's cpu cores count assigned to VM
  ram: Amount of MByte of RAM dedicated to the VM
  disks: List of disk entries
  - type: type of the image used for this disk like "raw", "qcow2", etc ...
    devname: device name, for example "hda"
    src: relative path to the installation path of the image, should be equals to any resource.unarchive's value
```

## Usage 
The `VM definition` is used by many roles of this collection but is mainly used as object accessible during the [rendering process](https://jinja.palletsprojects.com/en/3.0.x/templates/) of a used VM template which will output an XML with libvirt domain format used as VM definition by the libvirt API.

The scheme of a `VM definition` allow the user to define its custom target and platform properties whose combination is accessible by the template scope.

The `VM definition` object is used as variable parameter of the following roles for different use:
- `kvm_provision` role :
  - Each uri entry of `vm.metadata.sources` :
    - is fetched using [ansible.builtin.get_url](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/get_url_module.html) module which supports HTTP, HTTPS, or FTP URLs
    - is extracted if the `.unarchived` 's filename is different than the `.uri`'s filename
  -  will render the template defined by the `vm.metadata.template` file (otherwise the default one) by using the `VM definition` object accessible on its scope.
- `init_vm_connection` role :
  - will eventually look for auth variables inside the `vm.metadata.auth` and add them as ansible auth variables (ansible_user, ansible_password, etc ...).
    - **Note**: You can declare them instead as host/group variables adding the "ansible_" prefix to each property's name (for example ansible_user) however it depends by the connection plugin used to allow ansible to connect to the virtualized hosts. This alternative way is compatible with [ansible vault](https://docs.ansible.com/ansible/latest/user_guide/vault.html) used to encrypt credentials.
  - will eventually install its used libvirt network if `vm.net.type` is `network` (leveraging the creation to `libvirt_network` role) and add DHCP entry by using the `vm.net` and `vm.metadata.hostname` info.
  - will add the VM as ansible host to its inventory with alias `vm.metadata.name`, address of `vm.net.ip` and will add the host as member of the following ansible groups
    - `'vms'`
    - `vm.metadata.platform_name`
    - `vm.metadata.arch_name`
- `guest_provision` role
  - will use `vm.metadata.platform_name` and `vm.metadata.arch_name` to lookup the most specific available phase files. Custom variables may be also used by the user-defined phases.
