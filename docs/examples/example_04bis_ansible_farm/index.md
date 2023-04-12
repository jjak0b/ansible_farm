# Create a test farm and distribute VM

## Intro

In this example we are going show a real use case scenario by using some hypervisor targets and deploy some VMs on these based on their libvirt virtualization capabilities.
Each VM will be assigned to each hypervisor target by using the `vm_dispatcher` role and will be provisioned with simple application build to test if at least compile successfully on different architectures. This is a common scenario for continuous integration.

Note: In this example hypervisors will be bare metal hosts only but the concept of hypervisor target can be applied on any inventory host that has at least a libvirt environment, so you can build first a test farm made up of VM hypervisors and distribute the (nested) VM over these ones like explained in the [nested VM provisioning example](../example_03_nested_VM_provisioning_VDE/index.md). 

- [Full code](//github.com/jjak0b/ansible_farm/tree/master/docs/examples/example_04_test_farm/)

## Prerequisite

- [vm_dispatcher](../../roles/vm_dispatcher.md#Requirements ) role requirements
- [kvm_provision](../../roles/kvm_provision.md#Requirements ) role requirements
- [guest_provision](../../roles/guest_provision.md#Requirements ) role requirements

## Define the inventory

Let's define the inventory on different way:

- we have to specify the virtual machines configurations we want to generate on `localhost` (the ansible controller) but we won't it as hypervisor this time
- we have to specify some remote hypervisor hosts that will emulate the VMs, in this example will be the following ones assuming all requirements are already set up: 

  - `notebook_amd64`: is a notebook host with debian 11
  - `rpi_4_arm64`: is a Raspberry Pi 4 4GB with raspberry OS

```
all:
  hosts:
    localhost:
      ansible_connection: local
      config:
        permutations:
          targets:
            - 'amd64'
            - 'arm64'
            - 'ppc64'
          platforms:
            - 'debian_12'
        list:
          - platform: 'Arch-Linux'
            target: 'amd64'
  children:
    hypervisors:
      hosts:
        notebook_amd64:
          ansible_host: nb-jacopo.lan
          ansible_user: user
        rpi_4_arm64:
          ansible_host: 192.168.1.200
          ansible_user: pi
      vars:
        ansible_libvirt_uri: 'qemu:///session'
    Arch-Linux:
      vars:
        # arch image doesn't have python installed, so use ssh port-based waits instead
        wait_until_port_reachable: true
    arm64:
      vars:
        snapshot_type: external
    vms:
      vars:
        ansible_connection: ssh
        # ansible_host: localhost
        ansible_ssh_private_key_file: ~/.ssh/id_ssh_rsa_vm
        project_id: example_04
        project_revision: 0
        allowed_phases:
          - startup
          - restore init
          - create init
          - restore clean
          - create clean
          - dependencies
          - init
          - main
          - terminate
          - shutdown
```
Some notes are required:

- the `Arch-Linux` platform require `wait_until_port_reachable: true` since python is not installed but will be installed later, but until then we need to wait until SSH server is running.
- the `arm64` targets require to set a custom UEFI firmware but doing so [internal snapshots aren't supported by libvirt](https://unix.stackexchange.com/questions/663372/error-creating-snapshot-operation-not-supported-internal-snapshots-of-a-vm-wit), so we will use external snapshots instead by setting `snapshot_type: external`. 
- the `ansible_ssh_private_key_file: ~/.ssh/id_ssh_rsa_vm` wil be used as shared key eventually generated by the `setup_image.yaml` callback-task.

## Define targets

- **amd64**:
  ```
  - name: Define target definition
    set_fact:
      vm:
        metadata:
          target_name: amd64
        virt_domain: "{{ (ansible_architecture != 'x86_64') | ternary('qemu', 'kvm') }}"
        emulator: "/usr/bin/qemu-system-x86_64"
        arch: "x86_64"
        cpu: "{{ (ansible_architecture != 'x86_64') | ternary('qemu64', '') | default(omit, true) }}"
        machine: q35
  ```
- **arm64**
  ```
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
- **ppc64**
  ```
  - name: Define target definition
    set_fact:
      vm:
        metadata:
          target_name: ppc64
        virt_domain: "{{ (ansible_architecture != 'ppc64le') | ternary('qemu', 'kvm') }}"
        arch: "ppc64le"
        emulator: "/usr/bin/qemu-system-ppc64le"
        cpu: POWER9
        machine: pseries
        vcpus: 1
        default_bus: pci.0
  ```

Note: **arm64** require to set the firmware as previously mentioned and **ppc64** require to change the default bus used by the user network device with the `pci.0` bus, since the architecure doesn't support PCIe controllers.

## Define platforms

The platforms are defined with the usual way, you may look the example source code to check them.

We just show how to add the `debian_12.yaml` using a different image directly from debian and extend its size so we are able to install packages.

- Add a generic `setup_vm_/platforms/debian.yaml` to reuse for other Debian releases:
  ```
  - name: Defining platform with usermode networking
    vars:
      # version_name: bookworm
      # version_major_num: 12
      # version: "20230330-1335"
      period: daily
      vm_name: "VM {{ vm.metadata.platform_name }}_{{ vm.metadata.target_name }}"

      # Use this map to adapt custom target names to debian architecture aliases
      debian_arch_names:
        "arm64": "arm64"
        "amd64": "amd64"
        "ppc64": "ppc64el"
    block:

      - vars:
          ssh_forward_port: "{{ 65535 | random( start=2201, seed=vm_name) }}"
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
                      dest: "tmp-{{ image_name }}"
                    - callback: callbacks/sources/setup_image.yaml
                      src: "tmp-{{ image_name }}"
                      keys_path: "{{ vm.metadata.tmp_dir }}id_ssh_rsa_vm"
                    on_provision:
                      src: "tmp-{{ image_name }}"
                      dest: &image_file_name "{{ vm.metadata.platform_name }}_{{ image_name }}"
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
- add a specific `setup_vm_/platforms/debian_12.yaml` and insert a callback-task to increase image size:
  ```
  - block:
      - import_tasks: debian.yaml
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
  ```
- Add the `tasks/callbacks/sources/extend_and_convert.yaml` callback-task
  ```
  - name: preparing Image
    vars:
      src_path: "{{ (vm.metadata.tmp_dir, task.src) | path_join }}"
      dest_path: "{{  (vm.metadata.tmp_dir, task.dest) | path_join}}"
      src_path_raw:  "{{ (vm.metadata.tmp_dir, task.src) | path_join }}.img"
      from_format: "{{ task.from_format | default( 'raw', true ) }}"
      to_format: "{{ task.to_format | default( 'qcow2', true ) }}"
      partition_number: "{{ task.partition_number | default( 1, true ) | string }}"
      size: "{{ task.size }}"
    block:

      - name: convert to raw
        shell:
          cmd: >-
            qemu-img convert
            -f {{ from_format }} -O raw
            {{ src_path | quote }} {{ src_path_raw | quote }}

      - name: enlarge the disk image file
        shell:
          cmd: >-
            truncate --size={{ size }} {{ src_path_raw | quote }}
      
      - name: trick to update the GPT header
        shell:
          cmd: >-
            printf "fix\n" | /sbin/parted ---pretend-input-tty {{ src_path_raw | quote }} print
      
      - name: resize the root partition
        shell:
          cmd: >-
            /sbin/parted -s {{ src_path_raw | quote }} resizepart 1 100%

      - name: convert to chosen format
        shell:
          cmd: >-
            qemu-img convert
            -f raw -O {{ to_format }}
            {{ src_path_raw | quote }} {{ dest_path | quote }}

      - name: remove temp raw image
        file:
          path: "{{ src_path_raw }}"
          state: absent
  ```

## Define the playbook

The `main.yaml` playbook is defined like the usual way but we add the `vm_dispatcher` role tha will dispatch the VM configurations over the running hypervisors by adding them as `config.list` items of a `VM configuration` such that it will be parsed by the `parse_vms_definitions` role as usual.

```
---
- hosts: localhost
  connection: local
  gather_facts: yes
  
- name: Init VM on Hypervisor
  hosts: hypervisors
  gather_facts: yes

  roles:
    - vm_dispatcher
    - parse_vms_definitions

  tasks:  

    - name: init VM connection
      loop: "{{ virtual_machines }}"

      include_role:
        name: init_vm_connection
      
      loop_control:
        loop_var: vm

- name: VM provisioning on Hypervisor host
  hosts: vms
  gather_facts: no
  tasks:

    - block:
      
        - name: gather min facts of hypervisor host since some definitions require to use them
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

```

## Define the VM provisioning phases

Assume that we are working on developing a project, we can create a dedicate folder where to place that project dedicated provision phases e use dedicated custom variables every time we neeed to test or provision something.

In this example we assume we want to test the [VDE](https://github.com/virtualsquare/vde-2) project build process on the platforms and architectures we have just defined.

Create a custom `vars/vde.yaml` file where to put custom variables of the project:

```
phases_lookup_dir_path: provision_phases/VDE
project_id: example_04.vde
project_revision: 0
```

As you can deduce from the `phases_lookup_dir_path` variable we will define the provision phases in the `provision_phases/VDE` directory.
The step to add differents project are the same for different sub directories

Note: In this way we can select which project needs to run the provision phases by adding the `--extra-vars "@vars/vde.yaml"` parameter when running the playbook from command line.

- Now define the same platform-specific **dependencies** phase tasks we already used on previous examples to install a dependencies list

  - `provision_phases/VDE/debian_12/dependencies.yaml` (see example code)
  - `provision_phases/VDE/Arch-Linux/dependencies.yaml` (see example code)
  
    - Hint: In Arch-Linux python isn't pre-installed, so you need to install it first from raw command at the top of its `dependencies.yaml` 
    
      - ```
        - name: Ensure at least python is installed first
          raw: pacman -Syy python --needed --noconfirm
          become: yes
        ```
  - Debian dependencies list:

    ```
    dependencies:
      - git
      - build-essential
      - cmake
      - make
      - autogen
      - autoconf
      - automake
      - libtool
      - libpcap-dev
    ```

  - Arch-Linux dependencies list:

    ```
    dependencies:
      - git
      - base-devel
      - cmake
      - make
      - autogen
      - autoconf
      - automake
      - libtool
      - libpcap
    ```
- Now define the common `main.yaml` phase file for all platforms ( the VDE building process ):

  ```
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

## Run


```
ANSIBLE_CONFIG=ansible.cfg ansible-playbook main.yaml --extra-vars "@vars/vde.yaml"
```

Note: dependencies and this build process may change in the future  but is not a problem because this is the purpose of this test farm combined with ansible: test and verify builds on master branch.

Some possible future improvements of this test farm can be the integration of tests like unit tests, functional test and integration tests.