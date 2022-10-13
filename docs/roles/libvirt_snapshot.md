libvirt_snapshot
=========
Perform on the specified VM some operations like: lists, restore, create and delete snapshots

Requirements
------------

- [virsh](https://www.libvirt.org/manpages/virsh.html)

Role Variables
--------------

- `uri`: libvirt connection uri 
- `vm_name`: name of target VM
- `delete`: (optional) snapshot name to delete if any
- `create`: (optional) snapshot name to create if any
- `restore`: (optional) snapshot name to restore if any
- `list`: (optional) true if you want to list all snapshot names of the VM into the snapshot_list fact, nothing otherwise

Each optional operation register the command result into `snapshot_<operation>_result` registered var.

Note: if any subset of operations are specified, then the operations are executed in the following order:
1. `delete`
2. `create`
3. `restore`
4. `list`

Dependencies
------------

None

Example Playbook
----------------
```
- hosts: hypervisors
  vars:
    uri: "qemu:///system"
    vm_name: "My_VM_Name"
  tasks:
    - name: save cp1
      import_role: libvirt_snapshot
      vars:
        create: "my_checkpoint1"

    - name: do stuff

    - name: save cp2
      import_role: libvirt_snapshot
      vars:
        create: "my_checkpoint2"

    - name: lists my checkpoints
      import_role: libvirt_snapshot
      vars:
        list: true
    
    - name: show the my snapshots
      debug:
        var: snapshot_list

    - name: the role get the result from the std out
      debug:
        var: snapshot_list_result
      
    - name: remove last snapshot
      import_role: libvirt_snapshot
      vars:
        delete: "my_checkpoint2"

    - name: restore the first snapshot
      import_role: libvirt_snapshot
      vars:
        restore: "my_checkpoint1"
    
    - name: Delete directly specifying the tasks instead as alternative way
      import_role:
        name: libvirt_snapshot
        tasks_from: delete
      vars:
        snapshot_name: "my_checkpoint1"
    
```
License
-------

BSD

Author Information
------------------
None
