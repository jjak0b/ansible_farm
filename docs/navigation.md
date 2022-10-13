# jjak0b.deploy_farm Collection documentation

[Terms and data structure]()
    * [VM configuration](objects/vm_configuration.md)
    * [VM definition](objects/vm_definition.md)
[Roles]()
    * # Main roles 
    * [kvm_provision](../roles/kvm_provision/README.md)
    * [guest_provision](../roles/guest_provision/README.md)
    - - - -
    * # Recommended utility roles
    * [init_vm_connection](../roles/init_vm_connection/README.md)
    * [parse_vms_definitions](../roles/parse_vms_definitions/README.md)
    - - - -
    * # Others utility roles
    * [libvirt_network](../roles/libvirt_network/README.md)
    * [libvirt_snapshot](../roles/libvirt_snapshot/README.md)
[Examples]()
    * [Deploy a single VM](examples/01_deploy_single_vm.md)
    * [Deploy VMs using default template](examples/02_deploy_multiple_vms_default.md)
    * [Deploy VMs using custom network and templates](examples/03_deploy_multiple_vms_custom.md)
    * [Guest/VM provisionig](examples/04_guest_provisioning.md)
    