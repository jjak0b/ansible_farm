guest_provision
=========

This role provision VMs with custom VM lifecycle phases organized in phases. This role offer a revion-based cache for an **init** phase which may helpful to avoid long setup time.

### The VM Guest lifecycle
The lifecycle of the provisioned VM follows the following control flow:

- **Startup** phase
  1. Start the VM on its hypervisor;
  2. Wait until either timeout or VM is ready and reachable by using the specified connection method.
- **Init** and **dependencies** phases: Cache control
  1. if the "_init_" snapshot is available then restore it first;
  2. otherwise if the "_clean_" snapshot is available then restore it;
  3. otherwise create the "_clean_" snapshot.
  4. if the "_init_" snapshot hasn't been restored, then:
     1. Run **dependencies** use case tasks `{{ import_path }}/dependencies.yaml`
     2. Run **init** use case tasks `{{ import_path }}/init.yaml`
     3. Create "_init_" snapshot
- **Main** phase:
  1. Run **Main** use case tasks `{{ import_path }}/main.yaml`
- **Terminate** phase: Always execute even if any of the previous phases failed or succeeded
  1. Run **Terminate** use case tasks `{{ import_path }}/terminate.yaml`
- **Shutdown** phase
  - Try to shutdown VM gracefully or poweroff forcefully.

Where:

- The real "_init_" snapshot name is `{{ project_id }}.{{ project_revision }}.init` 
- `import_path` is a subpath that match with the most detailed phase file location, according to the target and platform type of the VM and that match with one of the following list path on this priority order which contains a phase file:

    1. `"{{ ( phases_lookup_dir_path, vm.metadata.platform_name, vm.metadata.target_name| path_join }}"`
    2. `"{{ ( phases_lookup_dir_path, vm.metadata.platform_name ) | path_join }}"`
    3. `"{{ phases_lookup_dir_path }}"`

Hint: The `import_path` is useful when some dependencies have different alias in some platform's packets manager, or user needs "ad hoc" tasks/vars for some others use cases.

A use case of `import_path` may needs specific tasks/vars for a target on platform or only platform; for instance you can use the following directory tree:

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

Note: All use case phase file like `dependencies.yaml`, `init.yaml`, `main.yaml` and `terminate.yaml` are optional, however a generic use case file as fallback is recommended.

Warning: You should manually change the `project_revision` variable value every time you want to use a different init or dependencies phase file because otherwise their tasks won't execute since the init phase would always be restored to the previous cached phase result.

Requirements
------------
- System running hypervisor:
  - Supported platform:
    - Theoretically any GNU/Linux distribution
    - Tested:
      - debian
  - libvirt environment
    - libvirt daemon active and running
- Packages
  - `sshpass`
    - optional but required to use password on ssh on vm connections
    - otherwise use [ansible vault](https://docs.ansible.com/ansible/2.8/user_guide/vault.html)
  - `libvirt-clients`
    - required by `guest_provision` role to handle snapshots using `virsh`

Role Variables
--------------

- `kvm_host`
  - required
  - it's the hypervisor's inventory_name that allow to delegate some tasks related to the hypervisor
- `vm`
  - required
  - It's the `VM definition` object required to specify which VM have to be provisioned

- `ansible_connection` (and its required parameters):
  - recommended
  - standard [connection plugins](https://docs.ansible.com/ansible/latest/plugins/connection.html)  are supported
      - `community.libvirt.libvirt_qemu` is supported
- `wait_until_reachable`: boolean (default: true)
  - if true will wait until VM becomes rechable in some way (by port or by connection) before run the VM provisioning, otherwise it won't wait.
- `wait_until_port_reachable`: boolean
  - required **only** if VM's platform doesn't ship with python or the `community.libvirt.libvirt_qemu` connection plugin isn't used
  - if `true` the ansible controller will wait until the VM's host can be reached at the provided `ansible_port` (or default port 22); otherwise python must be installed on the VM and the [wait_for_connection](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/wait_for_connection_module.html) module will be used.
- `connection_timeout`: (default: 120)
  - optional
  - Integer which indicates the max number of seconds the wait should last while trying the connection
- `retry_count`: (default: 10)
  - Integer which indicates the max count of attempts required to shutdown the VM
    - may be usefull combined with `retry_delay`
- `retry_delay`: (default: 5)
  - Integer which indicates the delay in seconds that elapse between a shutdown attempt and another (since a platform OS may require long times to shutdown properly)
    - may be usefull combined with `retry_count`
- `project_id`:
  - Project identifier which identify the `init`'s owner base for snapshots, since multiple project may share the same image base and re-use their respective init snapshots as cache.
- `revision_id`:
  - Revision identifier which identify "a change" of the `init` phase. if this value is different than the current active snapsnot's revision then the cache is considered as invalid and a reset of the `init` phase will occur, by restoring to the clean base and re-process the init phase.
    - You can interpret this as "a change version" or a sort of "hash" which "identify" the installed dependencies or the init phase's content.
- `allowed_phases`: ( defaults: all sub-phases ):
  - A list of (sub) phases of the lifecycle that are allowed to run, the unspecified phases are skipped. The values must be any subset of the following (unordered) list:
    - `startup`: will start the VM
    - `restore init`: will restore the init snapshot if possible
    - `create init`: will creare the init snapshot if possible
    - `restore clean`: will restore the clean snapshot if possible
    - `create clean`: will create the clean snapshot if possible
    - `dependencies`: will run the **dependencies** phase if possible
    - `init`: will run the **init** phase if possible
    - `main`: will run the **main** phase if possible
    - `terminate`: will run the **terminate** phase if possible
    - `shutdown` will shutdown the VM

Handlers
--------

- `shutdown_hypervisor`: You can notify this handler to shutdown the current VM's hypervisor if it's a VM.
  - **Note**: This will work only if the hypervisor is a VM handled by this collection since this may tracks the nested VMs in the current ansible instance, otherwise it does nothing.

Dependencies
------------

- Format of `VM definition` in `vm` var required by `roles/kvm_provision` and used in this role phase tasks
- Format of `VM definition` in `vm` var produced by `roles/parse_vms_provision`'s output and used in this role phase tasks

- `community.libvirt`
- `ansible.windows`
  - Only if using **Windows** (VM) targets

Example Playbook
----------------
```
- name: VM provisioning on Hypervisor host
  hosts: vms
  gather_facts: no
  serial: 1
  tasks:
  - block:
    - name: gather min facts if definitions use them
      setup:
        gather_subset: 
        - '!all'
    - name: "start KVM Provision role for '{{ vm.metadata.name }}'"
      include_role: 
        name: kvm_provision
    delegate_to: "{{ kvm_host }}"
    tags:
    - kvm_provision
  
  - block:
    - name: "Start VM provisioning of '{{ vm.metadata.name }}' "
      include_role: 
        name: guest_provision
    tags:
    - guest_provision
```

License
-------

GPL-3.0-or-later

Author Information
------------------

None
