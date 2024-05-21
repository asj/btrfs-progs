.. _Btrfs Device Assembly and Verification:

Btrfs supports volume management without any external configuration file. Let's look at the on-disk parameters that help bring independent devices together and how we handle the various ways in which it can confuse the dynamic assembling of devices and how to handle them.

To begin, `mkfs.btrfs` creates on-disk super-blocks `struct btrfs_super_block` and some basic root trees, which help the kernel validate the assembled devices at the time of mount. At this stage, these devices are read into the kernel using the `BTRFS_IOC_SCAN_DEV` ioctl at `/dev/btrfs/control`. Additionally, the command `btrfs device scan` can help to read all the Btrfs filesystem devices into the Btrfs kernel as well. The actual identification of the devices and their assembly as a volume happens inside the kernel.

The `list_head fs_uuids` in the kernel points to the list of all Btrfs fsids/filesystems in the kernel, with each one pointing to an fsid declared as `struct btrfs_fs_devices` (typically known as `fs_devices`). Furthermore, when all the devices are registered, the volume is formed at the `list btrfs_fs_devices::devlist`, which holds a linked list of `struct btrfs_device` to maintain the information for each device belonging to that filesystem.

Each of the devices in a Btrfs filesystem is distinguished using `btrfs_device::uuid` and `btrfs_device::devid`. The `btrfs_device::uuid` was unique in the kernel until kernel v6.7. The `temp_fsid` feature for single-device filesystems knows how to handle identical single devices.

So, when all the devices are registered, during the mount process, we need just one of the devices to mount the whole volume. However, if you prefer to specify the devices manually instead of using the kernel's automatic assembly, you can do so using the command `btrfs device scan --forget` to clear the kernel's known assembly, and then specify the devices in the mount option, `mount -o device=/dev/<dev1>,device=/dev/<dev2> <mnt>`.

Generation Number
-----------------

With the `struct btrfs_device::generation` number, we select the device that has the most recent transaction commit. Generally, in a healthy volume, all the devices will have the same generation number.

If there are multiple devices with the same matching fsid uuid and devid, the device with the larger generation number is always picked up. This is to avoid older or reappeared devices from being joined as part of the volume.

Once the devices are assembled, a device with the largest generation is picked by the mount thread to read the metadata at the root tree.

So far, we have identified the devices based on what each device declared through its super-block.

Now, let us look at how we verify each of these devices through the mount thread.

sys_chunk_array
---------------

As part of the struct btrfs_super_block, we also have an array of metadata chunks information defined as below:

.. code-block:: c

    #define BTRFS_SYSTEM_CHUNK_ARRAY_SIZE 2048
    btrfs_super_block::sys_chunk_array[BTRFS_SYSTEM_CHUNK_ARRAY_SIZE];

Each element of this array is of type CHUNK_ITEM and contains information about the metadata block group profile and the identification of those devices.

.. code-block:: bash

    sys_chunk_array[2048]:
      item 0 key (FIRST_CHUNK_TREE CHUNK_ITEM 22020096)
        length 8388608 owner 2 stripe_len 65536 type SYSTEM|DUP
        io_align 65536 io_width 65536 sector_size 4096
        num_stripes 2 sub_stripes 1
            stripe 0 devid 1 offset 22020096
            dev_uuid e9d99243-2b93-4917-9f5f-ed22507ec806
            stripe 1 devid 1 offset 30408704
            dev_uuid e9d99243-2b93-4917-9f5f-ed22507ec806

Additional devices that are required to join with other devices are listed in the system chunk array. The UUID and devid are taken from here and matched with the devices in the btrfs_fs_devices::devlist. Now, the device shall have the state BTRFS_DEV_STATE_IN_FS_METADATA. Only those devices where the metadata is placed are found here.

btrfs_read_chunk_tree
---------------------

The chunk-tree root is loaded from the btrfs_super_block::chunk_root and finds all the device items. The device items are of type struct btrfs_dev_item, from which the devices in the fs_devices::devlist are checked against devid, uuid, and fsid/metadata_uuid. The devices which fail to match are removed from the list. At this point, we also determine if there is any missing device and if there is a -o degraded option to override and continue with the degraded mount. If there is a missing device, a missing device added entry as a device with the expected devid and uuid as per the dev_item is added, and the rest of the devices get the BTRFS_DEV_STATE_IN_FS_METADATA state.

btrfs_verify_dev_extents
------------------------

Various device verifications, such as physical sizes, are done at this stage, including verification of dev extent to its chunk mapping.

btrfs_read_block_groups
-----------------------

To find out the block-group profiles being used, all the block groups are searched in the extent-tree or in a separate block-group-tree (since kernel v6.1). This provides us with a list of block groups that are already created. It can be visualized at:

    /sys/fs/btrfs/<fsid>/allocation/<type>/<bg>

For this reason, the mount time and the size of the extent tree were proportional, and it wasn't scalable on larger filesystems before v6.1. This issue was resolved by using a separate block-group-tree.

Missing device
--------------

Missing device identification happens at multiple levels. First, in the read_sys_array, where all the metadata-required devices are identified, and then in the chunk tree read, where all the device items are read, providing a complete list of all the devices. However, we don't yet know if we need all these devices to mount the volume in a degraded mode, which means to activate the RAID fault tolerance. This is determined when we read all the chunks in the filesystem chunk tree because these chunks can have different block-group profiles, and the number of devices required to reconstruct the data or to read from the mirror copy varies.

The missing device might reappear later, lacking the latest generation number. The filesystem will continue to work in a degraded state if the redundancy level allows. If it reappears, it shall be scanned; however, it won't join the allocation as of now. A mount recycle will be necessary following the balance so that the missing blocks on the missing device are copied.

Device paths
------------

During boot, we also allow the user to update the device path without going through the device open and close cycle because systems without the initrd shall use a temporary device path (/dev/root) for the initial bootstrap, which must be updated to the final device path when the system block layer is ready.

Also, at some point, we might mount a subvolume, in which case the device path is scanned again. So, it is essential to let the matched device path scan again and return success.
