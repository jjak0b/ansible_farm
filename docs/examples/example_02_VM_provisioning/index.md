Deploy multiple VM and provisioning them based on their platform
========================================================

Intro
-----

In this example we are going to deploy two VMs on a hypervisor host using and show how to provision the VMs with different tasks based on their platforms on a guest provision's phase (dependencies)

Note: The concept can be applied on any phase of the VM provisioning even if this example shows a use case of a single phase.

- [Full code](//github.com/jjak0b/ansible_farm/tree/master/docs/examples/example_02_VM_provisioning/)

Prerequisite
-------------
- Install `sshpass` in your hypervisor host
- [kvm_provision](../../roles/kvm_provision.md#Requirements ) role requirements
- [guest_provision](../../roles/guest_provision.md#Requirements ) role requirements

Define the inventory
--------------------

Let's define the ansible inventory by declaring a [VM configuration](../../objects/vm_configuration.md) that each hypervisor must manage with the VMs we need.

```
all:
  children:
    hypervisors:
      hosts:
        local_hypervisor:
          ansible_connection: local
          ansible_host: localhost
        vars:
          config:
            permutations:
              targets:
                - "amd64"
              platforms:
                - "debian_vs"
                - "Arch-Linux"
    # reuse same port of previous example
    debian_vs:
      vars:
        ansible_port: 2201
    vms:
      vars:
        should_setup_proxy_jumps: false
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

Define the main playbook
------------------------

Let's define the playbook [main.yaml](main.yaml) in different way than the [example_01](../example_01_basic_deploy_and_connect/index.md) such that if a VM host fail on any of hypervisor or guest provisioning phases, then it won't affects the provisioning of others VMs.

The following code allow to parse, generate the VM definitions and setup their connection such that each VM can be installed and provisioned later on an indipendent way.

```
---

- name: Init VM on Hypervisor
  hosts: hypervisors
  gather_facts: yes
  roles:
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
  serial: 1
  vars:
    project_id: example_02
    project_revision: 0
  tasks:

    - block:
      
        - name: gather min facts of hypervisor host since some definitions require to use them
          setup:
            gather_subset: 
            - '!all'
        
        - name: "start KVM Provision role for '{{ vm.metadata.name }}'"
          include_role: 
            name: kvm_provision
          vars:
            vm: "{{ vm }}"

      delegate_to: "{{ kvm_host }}"
    
    - block:
        - name: "Start VM provisioning of '{{ vm.metadata.name }}'"
          include_role: 
            name: guest_provision
```

Warning: The task "gather min facts ..." is required to do almost the same thing about `gather_facts: yes` but on a delegated host. We delegate the `kvm_provision` role to the VM's hypervisor since the role may require to auth to the hypervisor with specific host credentials or use a ansible_user that can access to libvirt to run the role tasks. These tasks like some custom resource callbacks may use some specific module which assume that some facts are already gathered by ansible (for example set the package manager of the host, or set a subset of facts). See the [setup module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/setup_module.html) for more details.



Define the provisioning phases
------------------------------

Now Let's suppose we want to define the **init**, **main**, and **terminate** phases which are common for all platforms we are going to deploy:

- `provisioning_phases/init.yaml`

  ```
  - name: A init phase task
    debug:
      msg:  "Inits something that could be cached"
  ```

- `provisioning_phases/main.yaml`

  ```
  - name: A main phase task
    debug:
      msg:  "Run something"
  ```

- `provisioning_phases/terminate.yaml`

  ```
  - name: A terminate phase task
    debug:
      msg:  "terminate something carefully"
  ```

 Let's suppose the dependencies we require to install are different or need different installation steps; So we need a different **dependencies** phase tasks specific for the two platforms previously defined:

 - `provisioning_phases/debian_vs/dependencies.yaml`
  
    ```
    - import_tasks: ../common-tasks/gather_sysinfo.yaml

    - name: Installing required dependencies
      vars:
        dependencies:
          - curl
      become: yes
      block:

        - include_tasks: ../common-tasks/sync-clock.yaml

        - name: Make sure the system is up to date.
          apt:
            update_cache: yes
            cache_valid_time: 3600
            state: latest
            upgrade: yes
          
        - name: Install dependencies.
          apt:
            update_cache: yes
            cache_valid_time: 3600
            name: "{{ dependencies }}"
            state: present

    ```
 - `provisioning_phases/Arch-Linux/dependencies.yaml`
  
    ```
    - name: Ensure at least python is installed first
      raw: pacman -Syy python --needed --noconfirm
      become: yes
    
    - import_tasks: ../common-tasks/gather_sysinfo.yaml

    - name: Installing required dependencies
      vars:
        dependencies:
          - wget
      become: yes
      block:

        - include_tasks: ../common-tasks/sync-clock.yaml

        - name: Make sure the system is up to date.
          pacman:
            update_cache: yes
            state: latest
            upgrade: yes
          
        - name: Install dependencies.
          pacman:
            update_cache: yes
            name: "{{ dependencies }}"
            state: present

    ```

Hint: If a phase task file for a specific platform doesn't exists we could provide a default `provisioning_phases/dependencies.yaml` as fallback since will be considered the most specific phase task file. However we won't use it in this example.

### Allow specific provisioning phases on VM

When we started this example we defined the `allowed_phases` variable in to inventory. That variable specify to the `guest_provision` role which (sub-)phases of the set should run. The order is not important since is considered as an unordered list. (Read `guest_provision` docs more details)

Hint: in this example we specified all the available phases but in some use cases ( for example testing while programming "on the fly"  ) may be very useful specify only a subset to save time.

Warning: Think to which possible consequences may happen (like missing snapshot restores) when you specify only a subset of them.


What happens ?
-----------------

If we run the `main.yaml` playbook, then for each VM definition: 

- will deploy the respective VM on its hypervisor
- will start the guest provisioning lifecycle

Note: Since the playbook has the `serial: 1` parameter set, then each VM will be processed on serial. If you want to process them on parallel then you need to remove that parameter.

Hint: The playbook of this example is almost identical to the `playbooks/main.yaml` of this collection so you can directly replace the `main.yaml` playbook content of this example with ``` - import_playbook: jjak0b.ansible_farm.main ``` if you installed the collection or use a symbolic link.

Let's focus on the provisioning lifecycle: all phases will execute in sequence.
The init phase should include all tasks of dependencies and init tasks that will install anything on the guest system and since the downloads and setup of dependencies may require a lot of time and resources, then it's often helful if it may cache that content and skip directly to the init phase. For such reason the `guest_provision` role will restore the image to the safest available snapshot 

1. if an "init" snapshot is available then will restore it first;
2. otherwise if a "clean" snapshot is available then will restore it;
3. otherwise will create the "clean" snapshot.

If the "init" snapshot has been restored then the provisioning lifecycle will skip directly to the main phase.
Otherwise if the "clean" snapshot has been either restored or created, then the init phase will be processed.

Hint: If you want to change this behavior you can specify which phase operations are allowed while provisioning by [setting the values into the `allowed_phases` variabe](../../roles/guest_provision.md#Role_Variables).

Let's conclude saying that:

On first run of `guest_provision` role:

- "debian_vs" platform will run:
  - `provisioning_phases/debian_vs/dependencies.yaml`
  - `provisioning_phases/init.yaml`
  - `provisioning_phases/main.yaml`
  - `provisioning_phases/terminate.yaml`
- "Arch-Linux" platform will run:
  - `provisioning_phases/Arch-Linux/dependencies.yaml`
  - `provisioning_phases/init.yaml`
  - `provisioning_phases/main.yaml`
  - `provisioning_phases/terminate.yaml`

but on a future runs of `guest_provision` role, assuming that the hypervisor won't re-install the VMs:

- "debian_vs" platform will run:
  - `provisioning_phases/main.yaml`
  - `provisioning_phases/terminate.yaml`
- "Arch-Linux" platform will run
  - `provisioning_phases/main.yaml`
  - `provisioning_phases/terminate.yaml`

### Run

Now in the terminal change the current directory to the main playbook's directory and run into the terminal: 

```
ANSIBLE_CONFIG=ansible.cfg ansible-playbook main.yaml
```
