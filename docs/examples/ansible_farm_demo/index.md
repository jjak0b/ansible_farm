# Ansible farm demo showcase

# Intro

This guide will cover a general setup for both Debian controller and managed target host up to use a use-case of this collection using KVM and QEMU hypervisors.

## Get demo in your ansible controller

```
git clone https://github.com/jjak0b/ansible_farm.git
cd ansible_farm/docs/examples/ansible_farm_demo
```

# Installation

## Setup Environment

The packages installation and group setup is the only part that require root privilages for this demo use case.


### Ansible controller host

Install required packages

```
su -
apt-get update
apt-get install ansible sshpass -y
exit
```

Install also git to download the collection and demo project

```
su -
apt-get install git
exit
```

Install the Ansible collection with

```
ansible-galaxy collection install git+https://github.com/jjak0b/ansible_farm.git,master
```

Install demo dependencies
```
ansible-galaxy role install stackhpc.libvirt-host
```


### Ansible managed host: the hypervisor host manual setup

Install required packages

```
su -
apt-get update
apt-get install --no-install-recommends -y libvirt-daemon-system libvirt-clients python3-libvirt python3-lxml
exit
```

In this demo we will need:

- QEMU KVM hypervisor
- qemu-img and virt-sysprep commands for preparing VM images

So install packages we need for this demo:

```
su -
apt-get install --no-install-recommends -y qemu-kvm qemu-utils libguestfs-tools
exit
```

This demo won't connect to `qemu:///system ` uri so we don't have to add the user to the `libvirt` group. Instead it will connect only to `qemu:///session ` uri but current user require to use KVM capabilities to speed up VM, so add it the `kvm` group

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

Prepare the images pool location where virtual machine disks will be stored

```
# Define the libvirt images pool
virsh pool-define-as --name default --type dir --target ~/.local/share/libvirt/images

# Create directoty tree, permission, etc ...
virsh pool-build default

# Start Pool
virsh pool-start default

# optional: enable autostart for pool
virsh pool-autostart default
```

#### Alternative hypervisor host setup: ansible way

You can alternatively setup the hypervisors using ansible **from the ansible controller host** to provision the hypervisors with all the previous hypervisor setup steps.

Note: that this will install **also all recommended requirements** of the collection.


Init demo project on controller:

```
git clone https://github.com/jjak0b/ansible_farm.git
cd ansible_farm/docs/examples/ansible_farm_demo
ansible-galaxy install -r requirements.yml
```

( Eventually add/update your  hypervisors -> L0 sub group inside the `hosts.yaml` ansible inventory file with your host(s) you want to use as hypervisor(s) )

Run the following playbook (change `become-user` and `become-method` with your needs ): 

```
ANSIBLE_CONFIG=ansible.cfg ansible-playbook playbook_00_setup_L0_hypervisors.yaml --ask-become-pass --become-user=root --become-method=su
```

## Use

