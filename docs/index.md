# jjak0b.ansible_farm Collection

An ansible collection used to create a farm of virtual machines and control these with tasks categorized in different provision phases.

The roles of this collection focus on:

- Create VM definitions from different configuration files separating platform-specific data and target-specific data
- Provision an hypervisor host with VMs of different VM definitions
- Provision a VM (guest) host with **repeateable** and **cachable** provision phases by using **snapshots**
- **Don't require root privileges** whenever possible and for the use main use cases of this collection don't require root privileges by default.

The main use case for this collection is to create a VM farm made of different platforms and targets to distribute repeateable and cachable provision phases over these like project builds and testing scripts.

This collection is flexible so you can use the features it provide also for other purposes.

All roles of this collection uses some common terms:

- The `VM configuration` is a convenient object which describes a permutations of all platforms and targets pairs that we need as virtual machines and each of those are going to be generated in the form of `VM definition` object after parsing some `platform and target definitions` files.
- The `VM definition` object describes the characteristics of a VM about the hardware it emulates, the firmware or OS that is installed onto it and other parameters like credentials and network host configuration. It's a combination of `platform` and `target` definitions.
- The `platform` definition (synonym of OS) is the definition of the used OS, disks, network, credentials, VM components that may be required by the OS. This specify also how a resource should be processed and installed later into libvirt.
- The`target` definition (synonym of machine, architecture of machine) is the definition of emulated hardware components in each `VM definition`, like CPU, RAM, machine type, emulator, etc ...

The Hypervisor and VM provisioning
----------------------------------

the `VMs configurations` should be defined in the hypervisors inventory and these configurations vars should be then provided as input of the `parse_vms_definitions` role such that it generates the `VM definitions` items in a `virtual_machines` list.

These `VM definitions` should be prepared with the `init_vm_connection` role first, installed using the `kvm_provision` role later and provisioned using the `guest_provision` role finally

### Standard Usage and How it works

- For each host of hypervisors
  - ( optional: Assign `VMs configurations` to the hypervisor host using `roles/vm_dispatcher` )
  - Provide all `VMs configurations` as input of `roles/parse_vms_definitions`
    - This will generate a list of `VM Definition` items called `virtual_machines`
  - Each `vm` item of `virtual_machines` should be provided as input of:
    - `roles/init_vm_connection`
      - to add a new ansible inventory host entry to the global inventory
      - to configure the connection to allow the controller to connect to the VM
        - Eventually defining a libvirt network and a DHCP entry for connection such that the vm should be connected to it
      - After that
        - each VM host is added as ansible inventory host
        - the `VM definition` is added as `vm` inventory host var
        - the hypervisor's `inventory_hostname` is added as `kvm_host` inventory host var to keep a reference of its hypervisor node.
        - Each VM host are added to the following ansible groups:
          - `vms`
          - `"{{ vm.metadata.name }}"`
          - `"{{ vm.metadata.platform_name }}"`
          - `"{{ vm.metadata.target_name }}"`

- For each host in `vms` should run:
  - Delegated `roles/kvm_provision` to `kvm_host`, to define and install the `VM definition` stored in `vm` inventory host var
  - `roles/guest_provision` to provision the VM with the guest lifecycle

The VM Guest lifecycle
----------------------

The lifecycle of the provisioned VM runs the following workflow:

0. **Startup** 
   - Start the VM
   - Wait until connection is ready

1. **Init** use case phase
   - Restore to a '**init**' snapshot if exists
   - otherwise fallback to restore or create a '**clean**' snapshot and run the **init** phase:

      1. **dependencies** pre-phase
         - Run dependencies tasks (`{{ import_path }}/dependencies.yaml`)
      2. use case phase: 
         - Run init tasks `{{ import_path }}/init.yaml`
         - Create 'init' snapshot

2. **Main** use case phase: 
   - Run main tasks `{{ import_path }}/main.yaml`

3. **Terminate** use case phase: 
   - Run end tasks `{{ import_path }}/terminate.yaml` whether the main phase succeeds or fails

4. **shutdown**
   - Shutdown gracefully first the VM, otherwise force it
 
Where `import_path` is a subpath that match with the most detailited phase file location, according to the target and platform type of the VM.
The `import_path` is the one in the following priority list path which contains a phase file:
- `"{{ ( phases_lookup_dir_path, vm.metadata.platform_name, vm.metadata.target_name| path_join }}"`
- `"{{ ( phases_lookup_dir_path, vm.metadata.platform_name ) | path_join }}"`
- `"{{ phases_lookup_dir_path }}"`

**Why:** A use case may needs specific tasks/vars for a target on platform or only platform; for instance:

- *debian_11* folder (`vm.metadata.platform_name` value in `platforms/debian_sid.yml`)
    - *amd64* folder (`vm.metadata.target_namevalue )
      - tasks or vars files, ... specific for *amd64* targets in *debian_11* platforms
    - *arm64* folder (`vm.metadata.target_namevalue )
      - tasks or vars files, ... specific for *arm64* targets in *debian_11* platforms
- *fedora_36* folder (`vm.metadata.platform_name` value )
    - *amd64* folder (`vm.metadata.target_namevalue )
      - tasks or vars files, ... specific for *amd64* targets in *fedora_36* platforms
    - tasks or vars files, ... specific *fedora_36* platforms but any target
- tasks or vars files, ... generic for any platform and target which file does not exists with a specific `import_path` sub path

The `import_path` is useful when some dependencies have different alias in some platform's packets manager, or user needs "ad hoc" tasks/vars for some others use cases.

Support and Requirements
------------

Read the documentation of each role for specific role's requirements.
The following tables shows support and requirements for the full collection.

- Required requirements are minimal and allow the collection to work but you will need at least some of recommended requirements to use in most cases
- Recommended requirements are used inside some builtin templates, target definitions and callback-tasks for common use cases


### Ansible controller host

<table title="Ansible controller support">
  <thead>
    <tr>
      <th>Platform</th>
      <th>Support</th>
      <th>Tested</th>
      <th>Requirements</th>
    </tr>
  </thead>
  </tbody>
    <tr>
      <td>Any GNU/Linux distribution</td>
      <td>should work if ansible support them</td>
      <td></td>
      <td rowspan="2">
        <ul>
          <li>ansible >=2.10.8
            <ul>
              <li>See control node <a href="https://docs.ansible.com/ansible/2.10/installation_guide/intro_installation.html#control-node-requirements">requirements</a></li>
            </ul>
          </li>
          <li>python >=2.6</li>
          <li>sshpass</li>
        </ul>
      </td>
    </tr>
    <tr>
      <td>Debian 11, 12</td>
      <td>yes</td>
      <td>yes</td>
    </tr>
  </tbody>
</table>


### Hypervisor target host:

<table title="Hypervisor target hosts support">
  <thead>
    <tr>
      <th>Platform</th>
      <th>Support</th>
      <th>Tested</th>
      <th>Requirements</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Any GNU/Linux distribution and others</td>
      <td>should work if libvirt and an hypervisor driver is supported</td>
      <td>partial (No SELinux)</td>
      <td>
        <ul>
          <li>Required
            <ul>
              <li>Ansible managed node <a href="https://docs.ansible.com/ansible/2.10/installation_guide/intro_installation.html#managed-node-requirements">requirements</a></li>
              <li>Configured libvirt environment</li>
              <li>Configured SSH server</li>
              <li>Configured Hypervisor compatible with libvirt. 
                <p>Note: Only builtin templates and target definitions use KVM or QEMU so you can use the hypervisor you want and override the builtin ones if needed.</p>
              </li>
            </ul>
          </li>
          <li> Recommended hypervisors
            <ul>
              <li>KVM</li>
              <li>QEMU</li>
            </ul>
          </li>
        </ul>
        <details>
          <summary>Required Commands</summary>
          <ul>
            <li>virsh</li>
            <li>qemu-img (external snapshots only)</li>
          <ul>
        </details>
        <details>
          <summary>Recommended Commands</summary>
          <ul>
            <li>virt-sysprep</li>
            <li>qemu-system-<code>&ltarch&gt</code></li>
            <li>qemu-img</li>
            <li>unzip</li>
            <li>gzip</li>
            <li>bunzip2</li>
            <li>xz</li>
          </ul>  
        </details>
      </td>
    </tr>
    <tr>
      <td>Debian 11, 12</td>
      <td>yes</td>
      <td>yes</td>
      <td rowspan="2">
        <details>
          <summary>Required Packages</summary>
          <ul>
            <li>libvirt-daemon-system</li>
            <li>python3-libvirt</li>
            <li>python3-lxml</li>
            <li>libvirt-clients</li>
            <li>qemu-utils</li>
          </ul>
        </details>
        <details>
        <summary>Recommended Packages</summary>
        <ul>
          <li>libguestfs-tools</li>
          <li>qemu-kvm</li>
          <li>qemu-utils</li>
          <li>unzip</li>
          <li>bzip2</li>
          <li>gzip</li>
          <li>xz-utils</li>
        </ul>
        </details>
      </td>
    </tr>
    <tr>
      <td>Ubuntu 22.04 LTS</td>
      <td>should work</td>
      <td>no</td>
    </tr>
    <tr>
      <td>Arch Linux</td>
      <td>should work</td>
      <td>no</td>
      <td>
        <details>
          <summary>Required Packages</summary>
          <ul>
            <li>libvirt</li>
            <li>libvirt-python</li>
            <li>python-lxml</li>
            <li>qemu-img</li>
          </ul>
        </details>
        <details>
          <summary>Recommended Packages</summary>
          <ul>
            <li>guestfs-tools</li>
            <li>qemu</li>
            <li>qemu-img</li>
            <li>unzip</li>
            <li>bzip2</li>
            <li>gzip</li>
            <li>xz</li>
          </ul>
        </details>
      </td>
    </tr>
  </tbody>
</table>

**Note: QEMU or KVM are recommended** for the following reasons:

- The collection support `qemu:///session` URI by default only when the `VDE` and `user` (userspace connections) virtual networks types are supported by the hypervisor.
- **[Since libvirt 9.0.0](https://libvirt.org/news.html#v9-0-0-2023-01-16)** the support of `passt` as network interface backend for [userspace connections](https://libvirt.org/formatdomain.html#userspace-slirp-or-passt-connection) has been added but it's [unstable](https://gitlab.com/libvirt/libvirt/-/issues/462), and so the VM template will use the network type `user` with that backend **since libvirt 9.2.0** only.
- **Prior libvirt 9.2.0** The `VDE` and `user` virtual networks are supported only when custom network interface can be added via the XML libvirt template through the libvirt [QEMU namespace](https://libvirt.org/drvqemu.html#pass-through-of-arbitrary-qemu-commands) for now. So other hypervisors may require specific extra configuration like definining other VM XML template.
- the [SSH connection plugin](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/ssh_connection.html) support may be achieved with `qemu:///session` only when SSH port of the VM is reachable from the hypervisor. The `user` network interface built with the **QEMU** namespace allow to specify a port forward with the `hostfwd` option or alternativelly using port forward with `passt`; but the first one is not supported by libvirt XML format for other hypervisors, and the second one is not supported on libvirt versions prior than `9.2.0`.
- the [community.libvirt.libvirt_qemu connection plugin](https://docs.ansible.com/ansible/latest/collections/community/libvirt/libvirt_qemu_connection.html) is supported only for local (ansible controller) hypervisor and **only if** you pre-install the **QEMU Guest Agent** on VM OS. The use of `ssh+qemu` URIs has not been tested.


### VM target host:

There are very few particular requirements for the platform used for virtual machines

<table title="VM target hosts support">
  <thead>
    <tr>
      <th>Platform</th>
      <th>Support</th>
      <th>Tested</th>
      <th>Requirements</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>GNU/Linux based OS</td>
      <td>yes</td>
      <td>yes (No SELinux)</td>
      <td rowspan="3">
        <ul>
          <li>Ansible managed node <a href="https://docs.ansible.com/ansible/2.10/installation_guide/intro_installation.html#managed-node-requirements">requirements</a></li>
          <ul>
            <li>Configured SSH server (preinstalled in VM image)</li>
          </ul>
          <li>Support for <code>/bin/sync</code></li>
        <ul>
      </td>
    </tr>
    <tr>
      <td>Mac OS</td>
      <td>yes</td>
      <td>no</td>
    </tr>
    <tr>
      <td>Others Unix-like OS</td>
      <td>should work</td>
      <td>no</td>
    </tr>
    <tr>
      <td>Windows</td>
      <td>yes</td>
      <td>no</td>
      <td>
        <ul>
          <li>Ansible managed node <a href="https://docs.ansible.com/ansible/2.10/installation_guide/intro_installation.html#managed-node-requirements">requirements</a></li>
          <ul>
            <li>Configured SSH server (preinstalled in VM image)</li>
          </ul>
          <li>Requirements for <a href="https://learn.microsoft.com/sysinternals/downloads/sync">Sync</a> executable</li>
        <ul>
      </td>
    </tr>
  </tbody>
</table>

Dependencies
------------

The following collections are ansible dependencies of this collection's roles and can be installed with ```ansible-galaxy install -r requirements.yml```

  - [community.libvirt](https://galaxy.ansible.com/community/libvirt) 
  - [community.general](https://docs.ansible.com/ansible/latest/collections/community/general/index.html)
  - [community.crypto](https://docs.ansible.com/ansible/latest/collections/community/crypto/index.html)
  - [ansible.utils](https://docs.ansible.com/ansible/latest/collections/ansible/utils/index.html)
