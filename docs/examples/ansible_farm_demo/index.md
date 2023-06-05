# Ansible farm demo showcase

# Intro

This guide will cover a general setup for both Debian controller and managed target host (hypervisors) up to use a use-case of this collection using KVM and QEMU hypervisors.
The use case demo showcase will be covered through a top-down approach so you can focus on results and use-case first and project development details later. 

# Installation

## Collection and libvirt environment setup

The packages installation and group setup is the only part that require root privilages for this demo use case.

### Ansible controller host

Install required packages

```
su -
apt-get update
apt-get install ansible sshpass -y
exit
```

Install also git to clone and install the collection

```
su -
apt-get install git
exit
```

Install the Ansible collection

```
ansible-galaxy collection install git+https://github.com/jjak0b/ansible_farm.git,master
```

### Ansible managed host: the hypervisor host manual setup


Install required packages

```
su -
apt-get update
apt-get install --no-install-recommends -y libvirt-daemon-system libvirt-clients qemu-utils python3-libvirt python3-lxml
exit
```

This demo will use **QEMU/KVM** hypervisors, so install the following required meta-package which will install the best `qemu-system-<arch>` package matching with your host architecture (usually `qemu-system-x86`)

```
su -
apt-get install --no-install-recommends -y qemu-kvm
exit
```

Note: If your project will connect to `qemu:///system ` uri you have to add the user to the `libvirt` group, otherwise you can connect `qemu:///session` uri without adding the user to any group. For more details see the libvirt documentation according to your system.

Prepare the images pool location where virtual machine disks will be stored if it doesn't exist

```
# Define the 'default' libvirt images pool
virsh pool-define-as --name default --type dir --target ~/.local/share/libvirt/images

# Create directoty tree, permission, etc ...
virsh pool-build default

# Start Pool
virsh pool-start default

# optional: enable autostart for pool
virsh pool-autostart default
```

Note: the libvirt pool directory needs to be accessible by the user that will be used to connect into the hypervisor host through ansible. The directory and images need rw permission by the `libvirt-qemu` group or `libvirt` group instead if you are using other not user-owned paths like `/var/lib/libvirt/images/`.

Hint: I recommend to use `~/.local/share/libvirt/images/` path for `qemu:///session` uri and `/var/lib/libvirt/images/` path + user as member of `libvirt` group for `qemu:///system` uri.

#### Enable and setup KVM 

Hint: if you want to speed up the VM then your user require to use KVM capabilities so add the user to the `kvm` group

```
su -

# Add kvm group if missing
groupadd kvm

# add user to kvm group
adduser <youruser> kvm

exit
```

Enable KVM and nested KVM feature on bare metal hypervisor

```
su -

# enable kvm
modprobe kvm

# enable nesting kvm on Intel bare metal host
modprobe kvm_intel nested=1 # use kvm_amd instead for AMD bare metal host

# enable kvm and nested kvm on boot
echo "options kvm_intel nested=1" >> /etc/modprobe.d/kvm.conf # or kvm_amd 
echo "options kvm" >> /etc/modprobe.d/kvm.conf
```

## Demo Installation

### Ansible controller host

Setup project on controller: install the ansible collection, roles and the demo playbooks

```
git clone https://github.com/jjak0b/ansible_farm.git
cd ansible_farm/docs/examples/ansible_farm_demo
ansible-galaxy install -r requirements.yml
```

The `requirements.yml` contains ansible collection and role dependencies for this demo :
- The `ansible_farm` collection
- The `stackhpc.libvirt-host` role used as utility to setup the libvirt environment, images pools and some network types.

### Ansible managed (hypervisor) host 

This demo will use

- QEMU/KVM hypervisor
- `qemu-img`, `virt-sysprep` and `cloud-localds` commands for preparing VM images

So install these other following packages on the hypervisor host

```
su -
apt-get install --no-install-recommends -y libguestfs-tools cloud-image-utils
exit
```

### ( Optional / alternative ) hypervisor host setup - The Ansible way

If you didn't setup the hypervisor hosts, you can alternatively use an ansible playbook **from the ansible controller host** to provision the hypervisors with all the previous managed hypervisor setup steps.

Note: that this will install the collection requirements and the demo requirements but **also all recommended requirements** of the collection but won't setup nested KVM feature.

1. add/update your hypervisors in the inventory: Update the host entries such that will be members of the hypervisors -> L0 sub group defined in the `hosts.yaml` ansible inventory file with your host(s) you want to use as hypervisor(s)

2. Create the playbook `playbook_00_setup_L0_hypervisors.yaml` to provision all the hypervisor requirements and recommendations.

    ```yaml
    - name: Setup bare metal L0 hypervisors
      hosts: "{{ hypervisors_group | default('hypervisors:&L0') }}"
      gather_facts: yes
      roles:
        - hypervisor_provision
    ```

3. Run the playbook and change the `become-user` and `become-method` values with your needs since it will require root privilages

    ```
    ansible-playbook -i playbook_00_setup_L0_hypervisors.yaml --ask-become-pass --become-user=root --become-method=su
    ```

# Demo showcase

This demo will show a Continuous Integration use case which will trigger a build and test workflow but we will focus on the farm provisioning to allow any test suite to be used from the provisioning phases orchestrated by this collection's features.

This demo is very similar to the [example 04](../example_04_test_farm/index.md) and will create a farm for the CI workflow for the [VDE project](https://github.com/virtualsquare/vde-2).

This demo is supposed to be used from a single amd64 bare-metal host which will deploy the farm structure through multiple playbooks in sequence, but the same purpose can be obtained by using multiple hosts as hypervisors to distribute the provisioning jobs.

## Ansible inventory

The inventory contains the variables of each managed node which can be organized in groups or entity types.

Now we need to model involved entities into Ansible groups and hosts: the controller, the hypervisors and the VMs.
So name these groups as 

- `localhost`
- `hypervisors`
- `vms`

The hypervisors can be

- Bare metal hosts, also called *L0* hypervisors
- other nested virtual machines, called *L1* hypervisors

So name these other groups as `L0` and `L1`

The virtual machines run a platform on an emulated target.
So Let's define which platforms and targets we want to deploy:
This depends by the use case but suppose we want to deploy/test the project in *Debian sid*, *Fedora 38* for both *amd64* and *arm64* architecture targets and *Arch Linux* only for *amd64* targets.

So name these respectively as 

- `debian_sid`, `fedora_38` and `archlinux`
- `amd64` and `arm64`.

We can model the configuration using a variable to define a `VM configuration`: 

```yaml
config: 
  permutations:
    platforms:
     - debian_sid
     - fedora_38
    targets:
     - amd64
     - arm64
  list: 
    - platform: archlinux
      target: amd64 
```

If we assign this configuration to a single hypervisor, then these VMs may run through emulation, para-virtualization or fully virtualization but a single bare metal PC hardware may not be able to handle that so we may need to dispatch these configurations to some other hypervisor hosts which can manage them. This job may be done by using the `vm_dispatcher` role if we assign the configuration to the `localhost` controller as host variable.

Now if you model the scenario just described we obtain the following ansible inventory  file `hosts.yaml`

```yaml
all:
  hosts:
    localhost:
      ansible_connection: local
      config:
        permutations:
          targets:
            - 'arm64'
            - 'amd64'
          platforms:
            - debian_cloud_sid
            - fedora_38
        list:
          - platform: archlinux
            target: 'amd64'
    
    local_hypervisor:
      ansible_connection: local
      ansible_host: localhost
  vars:
    ansible_libvirt_uri: 'qemu:///session'
  children:
  
    hypervisors:
      hosts:
        local_hypervisor:
      children:
        L0:
          hosts:
            local_hypervisor:
          vars:
        L1: 
          hosts:
          vars:
    vms:
      children:
        L1:
          vars:
            connection_timeout: 300
      vars:
        ansible_connection: ssh
        ansible_ssh_private_key_file: ~/.ssh/id_ssh_rsa_vm
    
    # vars specific per platform and target
    archlinux:
      vars:
        # arch image doesn't have python installed, so use ssh port-based waits instead
        wait_until_port_reachable: true
    arm64:
      vars:
        # We need to set the firmware but doing so internal snapshots aren't supported
        snapshot_type: external
```

As you can see you can use these groups to apply some variables or different connection variables to specific host and groups.

Note: A host like A nested VM is a member of both `hypervisors` and `vms` groups but also about `L1` group. This will be very usefull when we use the ansible playbooks.

## Playbooks

The Ansible playbooks describe how some jobs should be done and we use Ansible to delegate (or better orchestrate) some tasks to specific hosts, but Ansible allows to target also [specific patterns](https://docs.ansible.com/ansible/latest/inventory_guide/intro_patterns.html) using groups.

Before continue, there are 2 important playbooks that this collection provide since may be often used in most common use cases and so are provided as collection builtin: `init_vm_connection.yaml` and `run_vm_provision.yaml`

- `init_vm_connection.yaml` parse and init the pre-assigned VM configuration on their respective hypervisors.

  ```yaml
  - name: Init VM on Hypervisor
    hosts: "{{ hypervisors_group | default('hypervisors') }}"
    gather_facts: yes
    roles:
      - jjak0b.ansible_farm.parse_vms_definitions
    tasks:  

      - name: init VM connection
        loop: "{{ virtual_machines }}"

        include_role:
          name: jjak0b.ansible_farm.init_vm_connection
        
        loop_control:
          loop_var: vm
  ```

- `run_vm_provision.yaml` deploy the VM and run the provisioning phases on the running VMs.

  ```yaml
  - name: VM provisioning on Hypervisor host
    hosts: "{{ vms_group | default('vms') }}"
    gather_facts: no
    serial: 1
    tasks:

      - block:
        
          - name: gather min facts of hypervisor host since some definitions require to use them
            setup:
              gather_subset: 
              - '!all'
          
          - name: "start KVM Provision role for '{{ vm.metadata.name }}'"
            include_role: 
              name: jjak0b.ansible_farm.kvm_provision
          
        delegate_to: "{{ kvm_host }}"
        
        tags:
          - kvm_provision

      - block:

          - name: "Start VM provisioning of '{{ vm.metadata.name }}' "
            include_role: 
              name: jjak0b.ansible_farm.guest_provision
          
          - name: Flag VM as processed
            add_host:
              name: "{{ inventory_hostname }}"
              groups:
                - processed
        tags:
          - guest_provision

  ```

  Note: the `serial: 1` ansible parameter can be removed if you want to run the VMs in parallel.

  Note: The `gather_facts: no` ansible  parameter is required since the VMs may not may not have started until the `guest_provision` boot them


### The Farm structure 

This demo will use some playbooks to deploy a VM and provision it with a libvirt and QEMU/KVM hypervisor using the alternative hypervisor setup ([the Ansible way](#The_Ansible_way)) for the completeness of showing the collection features.

After the deploy that host will be added to the `hypervisors` group and will be a candidate during the VM assignement through the `vm_dispatcher` role.
The configuration of the eventually nested VM's SSH connection will be handled by the `init_vm_connection` whatever its hypervisor is.

We can now assume that there are some hypervisors that can handle the future farm.

Let's describe the farm structure we want build. 
We want to deploy and provision a VM to become a KVM hypervisor so we can deploy the VM and run the provisioning phases for a custom `hypervisor` project, but that VM must not turn off because must be available for other VM deployments. We can achieve this result with the following plays:

```yaml
- name: Init inventory with a L1 VM on the first reachable L0 hypervisor
  # import_playbook: jjak0b.ansible_farm.init_vm_definitions.yaml # From ansible 2.11
  import_playbook: init_vm_definitions.yaml
  # run only on the first reachable hypervisor
  run_once: true
  vars:
    hypervisors_group: "hypervisors:&L0"
    # Just init another debian VM on first reachable hypervisor
    config:
      list: 
        - target: amd64
          platform: debian_12

- name: Add all initialized VM to L1 and hypervisors group
  hosts: vms:!processed
  # gather_facts: no is important since vms don't exist yet
  gather_facts: no 
  tasks:

    - name: Add host to L1 group
      add_host:
        name: "{{ vm.metadata.name }}"
        groups:
          - hypervisors
          - L1

- name: Create the L1 VMs and make it an hypervisor but dont shutdown them
  # import_playbook: jjak0b.ansible_farm.run_vm_provision.yaml # From ansible 2.11
  import_playbook: run_vm_provision.yaml
  vars:
    # vms_group: "VM debian_sid_arm64"
    vms_group: "vms:&L1:!processed"
    # run provision phases to make them hypervisors
    phases_lookup_dir_path: provision_phases/hypervisor
    project_id: example_demo.hypervisor
    project_revision: 0
    allowed_phases:
      - startup
      # DO NOT use snapshots
      # - create clean
      # - create init
      # - restore init
      # - restore clean
      - dependencies
      - init
      - main
      # DO NOT SHUTDOWN: run L2 VM provisioning first
      # - terminate
      # - shutdown
```

Now we need to distribute VMs and add the plays that will run the VM provisioning lifecycle on the same ansible playbook instance and provision the `VDE` CI use case.

```yaml
- name: Assign and deploy some VMs over all hyperviors based on their capabilities
  hosts: hypervisors
  gather_facts: yes 
  vars:
    # Use this uri to check capabilities
    ansible_libvirt_uri: 'qemu:///session'
  roles:
    - jjak0b.ansible_farm.vm_dispatcher

- name: Init inventory with some L1 VMs and other L2 vms wherever they have been assigned to
  # import_playbook: jjak0b.ansible_farm.init_vm_definitions.yaml # From ansible 2.11
  import_playbook: init_vm_definitions.yaml
  vars:
    hypervisors_group: "hypervisors"


- name: Run some L1 or L2 VMs provisioning lifecycle wherever they have been assigned to
  # import_playbook: jjak0b.ansible_farm.run_vm_provision.yaml # From ansible 2.11
  import_playbook: run_vm_provision.yaml
  vars:
    vms_group: "vms:!processed"
    # run provision phases to make them hypervisors
    phases_lookup_dir_path: provision_phases/VDE
    project_id: example_demo.vde
    project_revision: 0
```

After this we can eventually stop the auxiliary L1 VM hypervisor and cleanup all the VMs with the following play:

```yaml
- name: "End L1 hypervisors lifecycle: VMs cleanup + Shutdown"
  # import_playbook: jjak0b.ansible_farm.run_vm_provision.yaml # From ansible 2.11
  import_playbook: run_vm_provision.yaml
  vars:
    vms_group: hypervisors:&L1
    phases_lookup_dir_path: provision_phases/hypervisor
    allowed_phases:
      - terminate
      - shutdown
    # dummy project name but wont be used
    project_id: example_demo
    project_revision: 0

- name: Delete all VMs
  hosts: vms
  tags:
    - delete
  gather_facts: no
  tasks:

    - name: Delegate vm Deletetion to respective hypervisor
      when:
        # Skip if hypervisor has been terminated
        - not ( kvm_host in groups['L1'] )
      block:
        
        - name: gather min facts of hypervisor host
          setup:
            gather_subset: 
              - '!all'
        
        - name: Cleanup VM owned by its hypervisor
          include_role:
            name: jjak0b.ansible_farm.kvm_provision
          vars:
            delete_vm: true
            create_vm: false
            should_remove_all_vm_storage: true

      delegate_to: "{{ kvm_host }}"
```

Hint: This last plays will undefine and delete VM assets and may need this behavior only in some cases, so when we run the playbook, we can append the `--skip-tags delete` flag on terminal to ignore the custom tasks/playbook with that assigned tag.


## Provision phases

We want create some jobs so we can build, test and eventually notify results.
The `guest_provision` role allow to perform repeatable tests and build leveraging ansible module idempotence combined with libvirt snapshots.

Project tests usually manage some parts: installation, initialization, test, and return results.

- Installations may be time consuming and require a lot of network bandwidth.
- inizializations may require same system configurations or project compilations which are often CPU intensive and time consuming
- tests may require to reset to a previous working checkpoint
- tests and their output may require some finalization steps before return the output to the user in any form and whenever the tests are successful or not
- All the previous contexts may require different jobs based on platform and target requirements.

The `guest_provision` role can make it easier to interact with these aspects: 

- Phases and snapshots allow cachable project installations and initializations
- Snapshots allow to create and reset tests to a custom checkpoint.
- phases allow to run specific platform (and target) tasks/jobs

Let's show an example of how to define some platform specific jobs for the `VDE` project.

Assume that we want to create a build job for some platform we already talk about.

#### Dependencies phase

Define the `dependency.yaml` file to handle the **dependency** phase: 
It will install some dependencies based on the VM 's platform info gathered by system info facts.

```yaml
- include_tasks: "{{ (phases_lookup_dir_path,  'common-tasks', 'gather_sysinfo.yaml' ) | path_join }}"

- block:

  - include_tasks: "{{ (phases_lookup_dir_path,  'common-tasks', 'sync-clock.yaml' ) | path_join }}"

  - name: Include dependencies using distribution defaults
    include_vars:
      file: "{{ ( 'dependencies', ansible_distribution + '.yml' ) | path_join }}"
    ignore_errors: yes
    when: dependencies is not defined

  - name: Update packet manager cache
    package:
      update_cache: true

  - name: Install dependencies.
    environment:
      DEBIAN_FRONTEND: noninteractive
    package:
      name: "{{ dependencies | default( [] ) }}"
      state: "{{ 'latest' if should_upgrade | default( false, true ) else 'present' }}"
  become: yes
```

- `provision_phases/VDE/common-tasks/gather_sysinfo.yaml` is an utility task that gather system info of the managed VM node like `gather_facts: yes` does on playbook level:

  ```yaml
  - name: Gather VM info
    setup:
      gather_subset: 
        - '!all'
  ```

- `provision_phases/VDE/common-tasks/sync-clock.yaml` is an utility task that make sure system clock is in sync with hardware clock to prevent time mismmatch caused by snapshot restores

  ```yaml
  - shell:
      cmd: /usr/sbin/hwclock --hctosys
  ```

Note: both the builtin ansible `setup` and `shell` module require to use **python** on the managed VM but in some VM images may not available, so you can install it.

This case happens with the `archlinux` platform: Define another **dependencies** task file at `provision_phases/VDE/archlinux/dependencies.yaml` and **install python using ansible modules that don't require python** and eventually include other dependency vars and the default **dependencies** task file

```yaml
- name: Ensure at least python is installed first
  raw: pacman -Syy python --needed --noconfirm
  become: true

- include_tasks: "{{ (phases_lookup_dir_path,  'common-tasks/gather_sysinfo.yaml' ) | path_join }}"

- block:
    - include_tasks: "{{ (phases_lookup_dir_path,  'common-tasks/sync-clock.yaml' ) | path_join }}"
  become: true

- name: Include dependencies using platform-specific dependencies
  include_vars:
    file: "{{ ( phases_lookup_dir_path, 'dependencies', ansible_distribution + '.yml' ) | path_join }}"
  ignore_errors: yes

- include_tasks: "{{ (phases_lookup_dir_path, 'dependencies.yaml' ) | path_join }}"
```

#### Init phase

The initialization phase is simple and common for all platforms but complex project that require very long compilation times the build process should be declared on the **init** phase, but VDE don't require too much to compile so we can declare the built job in the **main phase**

Note: The init phase is the last phase where snapshot are handled automatically so you can eventually restore to the init snapshot every time you need or create custom snapshots from the main phase.

#### Main phase

Create the `main.yaml`:

```yaml
- import_tasks: ./jobs/build.yaml
```

And create a build job at `provision_phases/VDE/jobs/build.yaml` :

```yaml
- name: Clone repo
  ansible.builtin.git:
    repo: 'https://github.com/virtualsquare/vde-2.git'
    dest: &prj_dir ~/VDE
    version: master
    recursive: true

- name: Resolve working directory
  ansible.builtin.stat:
    path: *prj_dir
  register: working_dir_result

- name: Init conf
  shell: 
    cmd: autoreconf -fis
    chdir: *prj_dir

- name: Configure
  shell: 
    cmd: ./configure --prefix=/opt/vde
    chdir: *prj_dir

- name: Build
  shell:
    cmd: make
    chdir: *prj_dir

- name: Install
  shell:
    cmd: make install
    chdir: "{{ working_dir_result.stat.path }}"
  become: yes

- name: Smoke test
  shell:
    cmd: /opt/vde/bin/vde_switch --version
  register: smoke_test_res

- debug:
    msg: "{{ smoke_test_res.stdout }}"
```

Note: the build job behaves in the same way of the [ci.yaml](https://github.com/virtualsquare/vde-2/blob/master/.github/workflows/ci.yml) github workflow of the official VDE project.


At this point we can suppose that we need a `jobs/test_01.yaml` that setup a test environment and some VDE features.
Let's suppose the `test_01.yaml` task ends and leave a dirty environment so we can restore to a previous `init` checkpoint (or a custom one) after the `test_01.yaml`:

```yaml
- import_tasks: ./jobs/build.yaml
- import_tasks: ./jobs/test_01.yaml

- name: restore the first snapshot
  import_role: jjak0b.ansible_farm.libvirt_snapshot
  vars:
    restore: "init"

- block:
    - include_tasks: "{{ (phases_lookup_dir_path,  'common-tasks/sync-clock.yaml' ) | path_join }}"
  become: true

- import_tasks: ./jobs/test_02.yaml
- import_tasks: ...
```

### Terminate phase

You can eventually save a report or set facts about the results or other stuff. This phase may be useful since is called whenever any task fail or the VM provision succeeded.

Note: If you restore snapshots You need to save your test report before the snapshot restore operation, otherwise you will lose it

For the VDE project we have no **terminate** phase for now.

## Platforms and targets

The platforms and targets we defined in the inventory VM configuration must be istanciated to VM definitions containing the virtual machines informations needed to tell libvirt which devices and components we need through the `kvm_provision` role.

Libvirt can't define some details like an image setup preprocessing, system credentials for user authentication, last image version checks, etc ...

A libvirt template is used to define the VM using some custom data and metadata defined in an istanciated VM definition, and other entries of the VM definition are used to declare **how** VM assets need to be elaborated first and installed later to work correctly.
Some of these important metadata are called *callback-tasks* which describe a task on an VM asset.

The `VM definitions` are variables or facts which are set from custom task files with the same filename of the platform's name and target's name.
These task files must define the VM definition portion which separate platform-specific and target-specific VM info such that the target definition may be re-used also for other VMs.
The platforms data are often dependent by the target but target definition data are common for most of platforms, so they are separated for reuse purpose.

- target definitions files allows to define a custom target with your alias name but which define components with official libvirt aliases for architecture names, devices, etc ...
- platform definition files allow to define a custom platform and adapt the custom target's name alias to the official architecture alias to use a correct VM image and override some target's definition details to adapt some platform's requirements if needed.
  - Example: Debian use the alias `amd64` and `arm64` for the `x86_64` and `aarch64` architecture names. Others distro may use different naming conventions.

The platform definition will be merged togheter with the target definition (eventually overriding some properties) and this define the `VM definition` instance that will be used for libvirt VM template, and in all roles of this collection.

### Targets

Let's show the target definitions:

- `setup_vm/targets/amd64.yaml`

  ```yaml
  - name: Define target definition
    set_fact:
      vm:
        metadata:
          target_name: amd64
        virt_domain: "{{ (ansible_architecture != 'x86_64') | ternary('qemu', 'kvm') }}"
        arch: "x86_64"
        emulator: "/usr/bin/qemu-system-x86_64"
        cpu: "{{ (ansible_architecture != 'x86_64') | ternary('qemu64', '') | default(omit, true) }}"
        machine: q35
  ```

- `setup_vm/targets/arm64.yaml`

  ```yaml
  - name: Define target definition
    set_fact:
      vm:
        metadata:
          target_name: arm64
        virt_domain: "{{ (ansible_architecture != 'aarch64') | ternary('qemu', 'kvm') }}"
        arch: "aarch64"
        emulator: "/usr/bin/qemu-system-aarch64"
        cpu: cortex-a53
        machine: virt
        vcpus: 1
        firmware: '/usr/share/AAVMF/AAVMF_CODE.fd'
  ```

Note: The virtualization domain type for the VM can be set through the `ansible_architecture` variable that depend by a previous `gather_fact` or `setup` module uses or through the `libvirt_supported_virtualizzation_types` variable provided by the `vm_dispatcher` role.

Warning: Ansible try to guess the virtualization type through the `ansible_virtualization_type` variable when using `gather_fact` or `setup` module but is not always correct, so i recommend to use the `libvirt_supported_virtualizzation_types` variable provided by the `vm_dispatcher` role.

### Platforms

Let's show the platform definitions of the L1 hypervisor VM:

- A parametric `setup_vm/platforms/debian_nocloud.yaml` define a generic Debian VM definition which download, and inject into the image the SSH public key and some scripts to configure users and SSH configurations on first boot using `virt-sysprep`

  ```yaml
  - name: Defining platform with usermode networking
    vars:
      # version_name: bookworm
      # version_major_num: 12
      # version: "20230330-1335"
      period: daily
      vm_name: "VM {{ vm.metadata.platform_name }}_{{ vm.metadata.target_name }}"
      debian_arch_names:
        "arm64": "arm64"
        "amd64": "amd64"
        "ppc64": "ppc64el"
    block:

      - vars:
          ssh_forward_port: "{{ hostvars[ vm_name ].ansible_port | default( 65535 | random( start=2201, seed=vm_name), true ) }}"
        block:
        
          - name: Set connection port for VM
            add_host:
              name: "{{ vm_name }}"
              ansible_port: "{{ ssh_forward_port }}"
        
          - name: Define platform definition
            vars:
              uri_base: "https://cdimage.debian.org/cdimage/cloud/{{ version_name }}/{{ period }}/{{ version }}"
              image_name: "debian-{{ version_major_num }}-nocloud-{{ debian_arch_names[ vm.metadata.target_name ] }}-daily-{{ version }}.qcow2"
            set_fact:
              vm:
                # metadata properties are used inside a task for installation and setup purposes
                metadata:
                  name: "{{ vm_name }}"
                  connection: "qemu:///session"
                  auth:
                    become_user: "root"
                    become_password: &root_pass "virtualsquare"
                    become_method: su
                    user: "user"
                    password: *root_pass
                  sources:
                  - before_provision:
                    - callback: callbacks/sources/fetch.yaml
                      src: "{{ uri_base }}/{{ image_name }}"
                      dest: "{{ image_name }}"
                    - callback: callbacks/sources/copy.yaml
                      src: "{{ image_name }}"
                      dest: &tmp_image_file_name "tmp-{{ vm.metadata.platform_name }}_{{ vm.metadata.target_name }}_{{ image_name }}"
                    - callback: callbacks/sources/setup_linux_image.yaml
                      src: *tmp_image_file_name
                      keys_path: "{{ vm.metadata.tmp_dir }}id_ssh_rsa_vm"
                    on_provision:
                      src: *tmp_image_file_name
                      dest: &image_file_name "{{ vm.metadata.platform_name }}_{{ vm.metadata.target_name }}_{{ image_name }}"
                vcpus: 2
                ram: 1536
                disks:
                  - type: "qcow2"
                    devname: "hda"
                    src: *image_file_name
                net:
                  type: user
                  source: "hostfwd=tcp:127.0.0.1:{{ ssh_forward_port }}-:22"
                  mac: "{{ '52:54:00' | random_mac( seed = vm_name ) }}"
                  # note: user networking is associated to hypervisor's ip
                  ip: "localhost"
  ```

- The specific `setup_vm/platforms/debian_12.yaml` override some part of the *debian_nocloud* definition like adding an image pre-processing callback-task to increase image size.

  ```yaml
    - block:
        - import_tasks:
            file: debian_nocloud.yaml
      vars:
        version_name: bookworm
        version_major_num: 12
        version: '20230403-1339'

    - vars:
        new_task:
          callback: callbacks/sources/extend_and_convert.yaml
          src: "{{ vm.metadata.sources[0].before_provision[2].src }}"
          dest: "{{ vm.metadata.sources[0].before_provision[2].src }}"
          from_format: qcow2
          to_format: qcow2
          partition_number: 1
          size: 10G
      block:

        - name: Add new task to imported definition
          set_fact:
            vm: "{{ vm | combine(vm_override, recursive=true ) }}"
          vars:
            vm_override:
              metadata:
                sources:
                  - before_provision: "{{ vm.metadata.sources[0].before_provision[:2] + [ new_task ] + vm.metadata.sources[0].before_provision[2:] }}"
                    on_provision: "{{ vm.metadata.sources[0].on_provision }}"
              ram: "{{ [ ( ansible_memfree_mb / 4 ) | int, 1024 ] | max }}"
              vcpus: "{{ [ ( ansible_processor_vcpus / 4 ) | int, 1] | max }}"
  ```

The *debian_sid* platform is almost identical to *debian_12* but different in the following parts:

- The downloaded image is compatible with the [cloud-init](https://cloud-init.io/) standard and so the platform replace the custom `setup_linux_image.yaml` callback-task with the custom `setup_cloudinit_iso.yaml` one that create an additional readonly ISO image disk through the `cloud-localds` command. This ISO will automatically configure the VM users, SSH and other system configuration when added to the VM.
- The builtin template does't allow to define readonly disks so this platform use the custom `cloud_compatible.xml.j2` template which is a copy of the default builtin template but with some changes to flag disks as readonly.

This is the generic `debian_cloud` platform definition: 

```yaml
- name: Defining platform with usermode networking
  vars:
    # version_name: bookworm
    # version_major_num: 12
    # version: "20230330-1335"
    period: daily
    vm_name: "VM {{ vm.metadata.platform_name }}_{{ vm.metadata.target_name }}"
    debian_arch_names:
      "arm64": "arm64"
      "amd64": "amd64"
      "ppc64": "ppc64el"
  block:

    - vars:
        ssh_forward_port: "{{ hostvars[ vm_name ].ansible_port | default( 65535 | random( start=2201, seed=vm_name), true ) }}"
      block:
      
        - name: Set connection port for VM
          add_host:
            name: "{{ vm_name }}"
            ansible_port: "{{ ssh_forward_port }}"
      
        - name: Define platform definition
          vars:
            uri_base: "https://cdimage.debian.org/cdimage/cloud/{{ version_name }}/{{ period }}/{{ version }}"
            image_name: "debian-{{ version_major_num }}-generic-{{ debian_arch_names[ vm.metadata.target_name ] }}-daily-{{ version }}.qcow2"
          set_fact:
            vm:
              # metadata properties are used inside a task for installation and setup purposes
              metadata:
                template: 'cloud_compatible'
                name: "{{ vm_name }}"
                connection: "qemu:///session"
                auth:
                  become_user: "root"
                  become_password: &root_pass "virtualsquare"
                  become_method: su
                  user: "user"
                  password: *root_pass
                sources:

                  - before_provision:
                      - callback: callbacks/sources/fetch.yaml
                        src: "{{ uri_base }}/{{ image_name }}"
                        dest: "{{ image_name }}"
                      - callback: callbacks/sources/copy.yaml
                        src: "{{ image_name }}"
                        dest: &tmp_image_file_name "tmp-{{ vm.metadata.platform_name }}_{{ vm.metadata.target_name }}_{{ image_name }}"
                    on_provision:
                      src: *tmp_image_file_name
                      dest: &image_file_name "{{ vm.metadata.platform_name }}_{{ vm.metadata.target_name }}_{{ image_name }}"
                  
                  - before_provision:
                      - callback: callbacks/sources/setup_cloudinit_iso.yaml
                        dest: &cloudinit_config_file_name "{{ vm.metadata.platform_name }}_{{ vm.metadata.target_name }}_cloud-config.img"
                        keys_path: "{{ vm.metadata.tmp_dir }}id_ssh_rsa_vm"
                    on_provision:
                        src: *cloudinit_config_file_name
                        dest: *cloudinit_config_file_name
              vcpus: 2
              ram: 1536
              disks:

                - type: "qcow2"
                  devname: "hda"
                  src: *image_file_name
                
                # MUST be declared after the rootfs. 
                - type: 'raw'
                  devname: 'hdb'
                  src: *cloudinit_config_file_name
                  # exclude from snapshots
                  readonly: true
              
              net:
                type: user
                source: "hostfwd=tcp:127.0.0.1:{{ ssh_forward_port }}-:22"
                mac: "{{ '52:54:00' | random_mac( seed = vm_name ) }}"
                # note: user networking is associated to hypervisor's ip
                ip: "localhost"
```

Others platforms definitions follow the same definition concept.

For more information about supported network types see the `init_vm_connection` [role](../../roles/init_vm_connection.md) and the VM definition [scheme](../../objects/vm_definition.md).


### Reuse farm structure in different projects

You can use the same farm structure for multiple projects if you create a parametric playbook that get the project's provision phases variables from other sources instead from static variables defined in the playbook with the [Ansible variables precedence rules](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html#variable-precedence-where-should-i-put-a-variable).

The [example 04 show this use case](../example_04_test_farm/index.md) by defining the variables that depends by the project in a separate variable `vars/vde.yaml` file:

```
phases_lookup_dir_path: provision_phases/VDE
project_id: example_demo.guest
project_revision: 0
```

And run the project with the shared farm structure with the following command:

```
ANSIBLE_CONFIG=ansible.cfg ansible-playbook main.yaml --extra-vars "@vars/vde.yaml"
```

# Conclusion

After a general showcase of what this collection can do i recommend to see also all provide examples which cover each role in depth with some explanations and details.

