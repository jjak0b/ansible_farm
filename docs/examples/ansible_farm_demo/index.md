# Ansible farm demo showcase

# Intro

This guide will cover a general setup for both Debian controller and managed target host (hypervisors) up to use a use-case of this collection using KVM and QEMU hypervisors.


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

Note: If your project will onnect to `qemu:///system ` uri you have to add the user to the `libvirt` group, otherwise you can connect `qemu:///session` uri without adding the user to any group. For more details see the libvirt documentation according to your system.

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

    ```
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

```
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

```
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

For the completeness of showing the collection features, this demo will use some playbooks to deploy a VM and provision it with a libvirt and QEMU/KVM hypervisor using the alternative hypervisor setup: [the Ansible way](#The_Ansible_way).




## Provision phases

TODO


