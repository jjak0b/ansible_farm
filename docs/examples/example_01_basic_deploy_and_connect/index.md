Deploy a VM and establish connection for VM provisioning
========================================================

Intro
-----

In this example we are going to show how to deploy a single VM on a hypervisor host using the default VM template and connect to it using SSH through the user network.

- [Full code](//github.com/jjak0b/ansible_farm/tree/master/docs/examples/example_01_basic_deploy_and_connect/)

Prerequisite
-----------------------

- [kvm_provision](../../roles/kvm_provision.md#Requirements ) role requirements
- Install `sshpass` in your hypervisor host because this example use credentials

Hypervisor provisioning
-----------------------

### Define the inventory: Add The hypervisor hosts

Define the hypervior host as localhost and using the local connection plugin inside the [hosts.yaml](hosts.yaml) file:

```
all:
  children:
    hypervisors:
      hosts:
        local_hypervisor:
          ansible_connection: local
          ansible_host: localhost

```

We are now going to define an ansible playbook which will deploy a VM on the hypervisor host.

The `kvm_provision` role will deploy the VM, but it requires a `vm` variable associated to its `VM definition` object which is going to be generated through the `parse_vms_definitions` role. You can define your custom `VM definition` but it must satisfy the minimial `VM definition` scheme (see the `VM definition` 's documentation for more details).

### Define a target

Let's define the [amd64](setup_vm/targets/amd64.yaml)  target definition which describe the architecture and the machine we are using, and store this as `setup_vm/targets/amd64.yaml`

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

### Define a platform

Let's define the platform [ debian_vs ](setup_vm/platforms/debian_vs.yaml)  which will use the image provided by [VirtualSquare](http://wiki.virtualsquare.org/#!/daily_brewed.md). Let's build the `VM definition` by defining it in to the `vm` fact and re-using some properties defined by the target in the `vm` fact, and store the following code as `setup_vm/platforms/debian_vs.yaml`:

```
- name: Defining platform with usermode networking
  vars:
    period: monthly
    version: "20220101-874"
    vm_name: "VM {{ vm.metadata.platform_name }}_{{ vm.metadata.target_name }}"
  block:
    - name: Define platform definition
      vars:
        uri_base: "https://www.cs.unibo.it/~renzo/virtualsquare/daily_brewed/{{ period }}"
        image_name: "debian-sid-v2-{{ vm.metadata.target_name }}-daily-{{ version }}.qcow2"
      set_fact:
        vm:
          # metadata properties are used inside a task for installation and setup purposes
          metadata:
            name: "{{ vm_name }}"
            hostname: "V2-{{ version }}"
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
                    src: "{{ uri_base }}/{{ image_name }}.bz2"
                    dest: &archive "{{ image_name }}.bz2"
                  - callback: callbacks/sources/unarchive.yml
                    src: *archive
                    dest: "./"
                on_provision:
                  callback: callbacks/sources/install_to_libvirt.yaml
                  src: "{{ image_name }}"
                  dest: &image_file_name "{{ vm.metadata.platform_name }}_{{ image_name }}"
          vcpus: 2
          ram: 2048
          disks:
            - type: "qcow2"
              devname: "hda"
              src: *image_file_name
          net:
            type: user
            source: "hostfwd=tcp:127.0.0.1:2201-:22"
            mac: "{{ '52:54:00' | random_mac( seed = vm_name ) }}"
            # note: user networking is associated to hypervisor's ip
            ip: "{{ ansible_host }}"

```

if you want to apply some common properties to all VM definitions you can define a (incomplete) `vm` fact into `defaults/platforms/defaults.yaml`: this will run just before each platform definition. If you aren't going to define it, the `parse_vms_definitions` role will use the one defined in its defaults.

Note: The `... fetch.yaml` and `... unarchive.yml` are utility callback-tasks to preprocess ( download, and unarchive ) the relative resource before install into the libvirt pool directory.

### Define a playbook: Deploy the VM into hypervisor

Let's define a [playbook_deploy.yaml](playbook_deploy.yaml) which will deploy a VM on the hypervisor host.

```
# Deploy only VM 
---

- name: Deploy a single VM on hypervisor
  hosts: hypervisors
  gather_facts: yes
  vars:
    my_vms_config:
      list:
        - platform: "debian_vs"
          target: "amd64"

  tasks:

    - name: Generate the VM definitions objects using my my_vms_config var as parameters
      import_role:
        name: parse_vms_definitions
      vars:
        config: "{{ my_vms_config }}"

    - name: Deploy a VM for each item in the 'virtual_machines' list
      loop: "{{ virtual_machines }}"
      
      include_role: 
        name: kvm_provision
      vars:
        vm: "{{ a_vm_definition }}"

      loop_control:
        loop_var: a_vm_definition

```

Note: Since we haven't defined a `vm.metadata.template`, then the `defaults/templates/default.xml.j2` template will be chosen when the `kvm_provision` role process. That default template is already provided into defaults of the `kvm_provision` role so we don't have to define a new one, but we must use some specific `vm` fact's properties it requires.


Connect to VM and VM provisioning
---------------------------------

The VM is using the user network and we defined a way to connect to it by using the `hostfwd` qemu's interface option. The property `vm.net.source: "hostfwd=tcp:127.0.0.1:2201-:22"` specify that the port 2201 port of the hypervisor's localhost is forwarded to the port 22 of the VM.

If we turn on the VM by using a frontend like `virt-manager` or `virsh` then we are able to reach the VM through SSH with: 

```
ssh user@localhost -p 2201 
```

But in ansible we need to define the connection info required by ssh to connect to the VM, and we are going to do this by defining them as ansible inventory host variables.


### Define the inventory: Add the VMs

The ansible variables required to connect to the VM depends by the `ansible_connection` plugin we chose to use.
If we chose `ssh` plugin, then they are the following:

```
ansible_connection: ssh
ansible_host: localhost
ansible_port: 2201
ansible_user: ...
ansible_password: ...
ansible_become_user: ...
ansible_become_password: ...
...
```

These variables can be defined in the:
- inventory host file 
- `host_vars/<my_vm_name>`.yaml file
- `group_vars/<my_vm_group_name>`.yaml file

And when the playbook will attempt to connect and perform some tasks on the VM then ansible will assign its respective variables on its VM scope (See [this](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#variable-precedence-where-should-i-put-a-variable) for details about this ansible feature ).

Note: If you use the user network interface on the VM then you should set:
- The hypervisor's IP/hostname as `ansible_host`
- The hypervisor's forwarded port as `ansible_port` ( `2201` in this example and the default one used by this collection if not overrided ) specified in the the `hostfwd` qemu's option.

Let's define these connection info inside the [hosts.yaml](hosts.yaml) inventory, by adding it for platform and target. It should looks like this:

```
all:
  children:
    hypervisors:
      hosts:
        local_hypervisor:
          ansible_connection: local
          ansible_host: localhost
    debian_vs:
      hosts:
        VM debian_vs_amd64:
      vars:
        ansible_connection: ssh
        ansible_host: localhost
        ansible_port: 2201
        ansible_user: user
        ansible_password: virtualsquare
        ansible_become_user: root
        ansible_become_password: virtualsquare

        # disable proxy jumps so we can use passwords
        should_setup_proxy_jumps: false

        # following are used by the guest_provision role
        project_id: example_01
        project_revision: 0
        allowed_phases:
          - startup

```

Hint: Since `parse_vms_definitions` is usually used to generate multiple VM, some platform dependent's info like the `ansible_user` may be useful to define them per platform group instead per host. This behavior may be achieved on runtime by the `init_vm_connection` role. It will add each VM as inventory host entry by using the `VM definition` object such that:

- The VM host entry will be a member of the following groups:
  - `vms` group ( general group containing all virtual machine entries )
  - `vm.metadata.platform_name` 's group ( `debian_vs` in this example )
  - `vm.metadata.target_name` 's group ( `amd64` in this example )
- The VM host entry will contains the following entries as host variables:
  - `kvm_host` will be associated to the **hypervisor**'s inventory hostname, which may useful to delegate some tasks directly to underling hypervisor.
  - `ansible_host` will be associated to the value of its `vm.net.ip` and this must be reachable by the ansible controller node.
  - `ansible_user`, `ansible_password`, `ansible_become_user`, `ansible_become_passwrd`, `ansible_become_method` will be associated to respective values of `vm.metadata.auth`'s properties (without 'ansible_' prefix).

If you use SSH authentication with credentials (this is the case of this example) you need to:

- Install `sshpass` in your hypervisor host
- set `should_setup_proxy_jumps: false` to disable authentication with keys and proxy jumps setup

Note: The `ansible_connection` and `ansible_port` values aren't set by any role of this collection and ansible will use its default ones if they are not overwritten ( they are usually `ssh` and `22` respectively ).

Hint: Alternatively you can assign a random (but repeatable) port that depends by your VM name so you can automatically assign a connection port to your VMs that use the same platform definition. So each VM have its own indipendent port since it depends by the VM's name. You can achieve this by adding the following code in your `setup_vm/<platform_name>.yaml`:

- Set a temp fact or variable: ```your_ssh_forward_port: "{{ hostvars[ vm_name ].ansible_port | default( 65535 | random( start=2201, seed=vm_name), true ) }}"```
- Set your ```vm.net.source: "hostfwd=tcp:127.0.0.1:{{your_ssh_forward_port}}-:22"```
- add the task:
  ```
  - name: Set connection port for VM
    add_host:
      name: "{{ vm.metadata.name }}"
      ansible_port: "{{ your_ssh_forward_port }}"
  ```

###  Define a playbook: VM provisioning

Now let's define the [playbook_connect.yaml](playbook_connect.yaml) that will connect and manage the VMs as following:


```
# Define connection to VM and start VM provisioning
---

- name: Fill ansible inventory with VMs
  hosts: hypervisors
  gather_facts: yes
  tasks:  
    - name: Init VM connections using the provided VM definition
      loop: "{{ virtual_machines }}"

      include_role: 
        name: init_vm_connection
      vars:
        vm: "{{ a_vm_definition }}"

      loop_control:
        loop_var: a_vm_definition

- name: VMs provisioning
  hosts: vms
  gather_facts: no
  serial: 1
  tasks:

    # Note: this is not covered by this example
    - name: "Handle the VM power on/off by delegating it to '{{ kvm_host }}' and start provisioning of '{{ vm.metadata.name }}' "
      include_role: 
        name: guest_provision

```

Warning: In the playbook "VMs provisioning" the `gather_facts` must be set to `no`/`false` because the VMs may not be ready to receive ssh connection yet but ansible will try to connect to them and if `gather_facts: yes` then it will run the [setup module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/setup_module.html), but will fail because VMs are unreachable until SSH service is running. The VM startup/shutdown and waits for the connection behaviors are handled by the `guest_provision` role.

Final steps
-----------
### Define a main playbook

Now we can just re-use the written playbook and import them in a single [main.yaml](main.yaml) playbook to run both the deploy and connect playbooks: 

```
- import_playbook: playbook_deploy.yaml

- import_playbook: playbook_connect.yaml
```

Note: **playbook_deploy** defines the `virtual_machines` fact which is accessible also by **playbook_connect**.

### Add ansible configuration

Now let's specify to ansible where to find roles, inventory, python and other parameters, by adding the following code to a [ansible.cfg](ansible.cfg) :

```
[defaults]
inventory = hosts.yaml
roles_path = ../../../roles/ 
interpreter_python = /usr/bin/python3

# Disable key checking for all hosts, but we usually want this only on VMs
host_key_checking = False
```

Warning: The `host_key_checking = False` setting is equivalent to the `ansible_ssh_common_args: '-o StrictHostKeyChecking=no'` variable. This ssh parameter is required to avoid that ansible pauses execution and prompts user to add the host to known_hosts. Otherwise this would happen every time ansible will attempt to connect to every new generated VM.

### Run

Now in the terminal change the current directory to the main playbook's directory and run into the terminal: 

```
ANSIBLE_CONFIG=ansible.cfg ansible-playbook main.yaml
```
