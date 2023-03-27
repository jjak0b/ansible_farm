libvirt_snapshot
=========

Perform some operations like: lists, restore, create and delete snapshots on a specified VM.
Both internal and external snapshots are supported. It's your responsibility to specifcy which type use by setting the `snapshot_type` variable.
- Internal snapshots
  - use the functionalities supported by `virsh`
- External snapshots
  - **create** and **list** operations use the functionalities supported by `virsh`
  - **restore** and **delete** operaions are "manually" emulated by using other features since they aren't supported by `virsh`

Note about the **restore** operation for _external snapshots_:
The feature try to emulate the same behavior of the restore operation for internal snapshots but **without restoring the memory state** :
 - shutdown (eventually) the VM 
 - for each disk in the snapshot's domain: create a new overlay disk and set it as current disk in the VM with the snapshot's disk as its backingStore
 - edit the snapshot definition to restore by updating the overlay disks in use, and set it as current.
 - restore the VM to the previous running state before the shutdown. ( note: this is just a poweron if it was running ).
 
 So the active disks will be the new overlay disks, with the backing disks of the restored snapshot.


Requirements
------------

- [virsh](https://www.libvirt.org/manpages/virsh.html) command
- `qemu-img` ( `qemu-utils` package ) command

Role Variables
--------------

These variables specify which operations should be done:
- `delete`: (optional) snapshot name to delete if any. Will delete also descedendants snapshots
- `create`: (optional) snapshot name to create if any
- `restore`: (optional) snapshot name to restore if any.
- `list`: (optional) true if you want to list all snapshot names of the VM into the snapshot_list fact, nothing otherwise

Each optional operation register the command result into `snapshot_<operation>_result` registered var.

Other operaion variables are the following:

- `uri`: the libvirt connection uri 
- `vm_name`: name of target VM / domain name
- `snapshot_type`: (default: `internal`) type of snapshot to operate on.
  - possible values:
    - `internal`
    - `external`
- `should_restore_with_new_branch`: (default: `false`) affect only if `snapshot_type == 'external'` and the `restore` or `delete` operation is specified
  - possible values:
    - `true` if you want to restore to the snapshot and create an overlay branch and setting it as current snapshot overlay and use new overlay disks.
    - `false` if you want to restore to the snapshot's domain (backing store domain), so after reboot the VM will use the backingStore disks as current disks.
- `should_delete_dandling_overlays`: (default: `true`) affect only if `snapshot_type == 'external'` and the `restore` or `delete` operation is specified
  - possible values:
    - `true` if you want to delete the previous overlay image **IF IT HAS NO DESCENDANTS SNAPSHOTS** (that depends by the specified snapshot), otherwise will keep it anyway.
    - `false` if you want to keep the dandling overlay image.
      Note that:
      - The overlay reference on the provided snapshot to restore will be lost anyway, since a new overlay image will be created.
      - This option should be only used if you want to manually re-use the dandling disk later for some reason.
  - Note:
    -  A dandling overlays is an an overlay image that is tracked by a direct descendant, but its parent (snapshot) doesn't point to it by its snapshot.
      - Graphic example:
        - before restore:
          ```
          parent snapshot <-> snapshot <-> overlay
          ```
        - after restore to "snapshot":
          ```
          parent snapshot <-> snapshot (current) <- overlay
                                                 <-> new overlay
          ```
Note: if any subset of operations are specified, then the operations are executed in the following order:
1. `delete`
2. `create`
3. `restore`
4. `list`

Dependencies
------------

- `community.general`
- `community.libvirt`
- `ansible.utils`


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
