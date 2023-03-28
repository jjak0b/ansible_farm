libvirt_snapshot
=========

Perform some operations like: **list**, **restore**, **create** and **delete** snapshots on a specified VM.
Both internal and external snapshots are supported. It's your responsibility to specify which type use by setting the `snapshot_type` variable.
- Internal snapshots
  - use the features supported by `virsh`
- External snapshots
  - **create** and **list** operations use the features supported by `virsh`
  - **restore** and **delete** operaions are "manually" emulated by using other features since they aren't supported by `virsh`

Hint: The use of `/bin/sync` command before the use of any **create** operation is recommended. That command isn't handled by this role. since this role focus to support snapshot features for hypervisor targets.

Note: The **restore** operation for _external snapshots_ feature try to emulate the same behavior of respective internal snapshots operations but **without restoring the memory state** . This is needed since external snapshots are only partially supported by `virsh` ( but [recent versions > 9.0 have more support](https://libvirt.org/news.html#v9-0-0-2023-01-16) ). This role add some support for specific snapshot restore and delete use case operations by updating the VM domain and some snapshot metadata, since `virsh` doesn't update the VM domain and snapshots automatically.

Warning: if you use a `restore` or `delete` operation with `snapshot_type == 'external'` variable, then after that task you should add a task with [wait_for_connection](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/wait_for_connection_module.html) module or any wait until VM is reachable before doing any action on the VM target since these operations require a VM restart to make changes effective.

If you don't know some specific terms about external snapshots i recommend you some readings before continue on using external snapshots:
- [External Snapshot management](https://wiki.libvirt.org/I_created_an_external_snapshot_but_libvirt_will_not_let_me_delete_or_revert_to_it.html)
- [Merging disk image image chains](https://libvirt.org/kbase/merging_disk_image_chains.html)

The **restore** _external_ operation depends by the `should_restore_with_new_branch` variable which has different behaviors:
  - it shutdown (eventually) the VM 
    - case `true`:
      - for each disk in the snapshot's domain
        - Create a new overlay disk (with random-generated name) and replace the previous snapshot overlay disk
        - Set the new overlay as a current disk in the VM domain
        - Set the disk of the VM domain saved in the snapshot as the new overlay's backing-store
      - **So the current disks in the VM domain are the new overlay disks, linked to the previous disks of the VM domain saved in the snapshot as their respective backing-store** (and nested backing-chain).
        - Graphic example of the situation
          ```
          parent snapshot <-> snapshot (restored) <-  previous overlay
                                                  <-> new overlay
          ```
    - case `false`:
      - Set the VM domain identical to the VM domain saved into the snapshot
      - **So the current disks in the VM domain reuse the same overlay disks of the snapshot with their respective backing-store**.
        - Graphic example of the situation
          ```
          parent snapshot <-> snapshot (restored) <->  previous = current overlay
          ```
  - edit the snapshot definition to restore by (eventually) updating the overlay disks in use, and set it as current.
  - restore the VM to the previous running state before the shutdown. ( note: this is just a poweron if VM was running ).

The **delete** _external_ operation focus to delete **a snapshot and its descendants** by deleting their metadata and deleting overlay disks from snapshot leaves up to the snapshot target and **behaves in different way than internal one**: it try to emulates the result of `virsh snapshot-delete` for each entry of `virsh snapshot-list --from <snapshot> --descendants --external`. This is a simplification for an easier management.

Warning: if the current VM snapshot should be affected by the deletion, then a `restore` operation will occur before the deletion to restore with the `should_restore_with_new_branch: false` variable parameter.

Note: In this case the use `should_restore_with_new_branch: false` is required as any current snapshot overlay disk will be deleted and in fact we just need to restore to the VM domain saved in the snapshot.

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
- `should_restore_with_new_branch`: (default: `true`) affect only if `snapshot_type == 'external'` and the `restore` or `delete` operation is specified
  - possible values:
    - `true` if you want to restore to the snapshot and create an overlay branch and setting it as current snapshot overlay and use new overlay disks.
    - `false` if you want to restore to the snapshot's domain (backing store domain), so after reboot the VM will use the backingStore disks as current disks.
- `should_delete_dandling_overlays`: (default: `true`) affect only if `snapshot_type == 'external'` and the `restore` or `delete` operation is specified
  - possible values:
    - `true` if you want to delete the previous overlay image **IF IT HAS NO DESCENDANTS SNAPSHOTS** (that depends by the specified snapshot), otherwise will keep it anyway.
    - `false` if you want to keep the dandling overlay image.
      **Note that:**
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
          parent snapshot <-> snapshot (current) <-  previous overlay (dandling)
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
