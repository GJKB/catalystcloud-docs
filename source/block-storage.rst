#############
Block storage
#############


********
Overview
********

Block volumes are similar to virtual disks that can be attached to any compute
instance in a region to provide additional storage. They are highly available
and extremely resilient.

Our block storage service is provided by a fully distributed storage system,
with no single points of failure and scalable to the exabyte level. The system
is self-healing and self-managing. Data is seamlessly replicated on three
different servers in the same region, making it fault tolerant and resilient.

The loss of a node or a disk leads to the data being quickly recovered on
another disk or node. The system runs frequent CRC checks to protect data from
soft corruption. The corruption of a single bit can be detected and
automatically restored to a healthy state.

Storage tiers
=============

b1.standard
-----------

The ``b1.standard`` tier combines SSDs with spinning drives to provide a good
balance between performance and cost. Writes are always done to SSDs first and
then flushed to HDDs later behind the scenes. Reads are likely to be cached by
the aggregate memory of all our storage nodes combined, but will hit a HDD when
the data is not cached.

Data stored on the ``b1.standard`` storage tier is replicated on three
different storage nodes on the same region.

Each ``b1.standard`` volume is limited to 1000 IOPS. You can stripe multiple
volumes together (using RAID 0) to achieve higher IOPS.

Additional storage tiers
------------------------

Catalyst is prepared to introduce additional storage tiers and is currently
waiting for demand from customers to introduce a faster tier backed purely by
SSDs. If you are interested and would like to see this available as soon as
possible, please contact your account manager.

Best practices
==============

The root volume of your compute instance should only be used for operating
system data. It is recommended to add additional volumes to your compute
instances to persist application data. For example: when running a MySQL
database, you should add at least one additional volume with enough space to
hold your database and mount it on ``/var/lib/mysql``.

While block volumes can be formatted and used independently, it is highly
recommended to use a logical volume management layer, such as LVM, in
production environments. By using LVM you will be able to add additional
volumes and resize file-systems without downtime. Please consult the
documentation of your operating system for information on how to use LVM.

If you are using volumes independently (without LVM, in a development
scenario), then you must label your partitions to ensure they will be mounted
correctly. The order of the devices (sdb, sdc, etc) may change and, when
unlabelled, may result in them being mounted incorrectly.



***********
Via the CLI
***********

Create a new volume
===================

Use the ``openstack volume create`` command to create a new volume:

.. code-block:: bash

  $ openstack volume create --description 'database volume' --size 50 db-vol-01
  +---------------------+--------------------------------------+
  | Field               | Value                                |
  +---------------------+--------------------------------------+
  | attachments         | []                                   |
  | availability_zone   | nz-por-1a                            |
  | bootable            | false                                |
  | consistencygroup_id | None                                 |
  | created_at          | 2016-08-18T23:08:40.021641           |
  | description         | database volume                      |
  | encrypted           | False                                |
  | id                  | 7e94a2f6-b4d2-47f1-83f7-a200e963404a |
  | multiattach         | False                                |
  | name                | db-vol-01                            |
  | properties          |                                      |
  | replication_status  | disabled                             |
  | size                | 50                                   |
  | snapshot_id         | None                                 |
  | source_volid        | None                                 |
  | status              | creating                             |
  | type                | b1.standard                          |
  | updated_at          | None                                 |
  | user_id             | 4b934c44d8b24e60acad9609b641bee3     |
  +---------------------+--------------------------------------+

Attach a volume to a compute instance
=====================================

Use the ``openstack server add volume`` command to attach the volume to an
instance:

.. code-block:: bash

  $ openstack server add volume INSTANCE_NAME VOLUME_NAME

The command above assumes that your volume name is unique. If you have volumes
with duplicate names, you will need to use the volume ID to attach it to a
compute instance.


*************
Using volumes
*************

Once attached to a compute instance, a block volume behaves like a raw
unformatted disk.

On Linux
========

The example below illustrates the use of a volume without LVM.

.. warning::

  Please note that this configuration is not suitable for production servers,
  but rather a demonstration that block volumes behave like regular disk drives
  attached to a server.

Check that the disk is recognised by the OS on the instance using ``fdisk``:

.. code-block:: bash

  $ sudo fdisk -l /dev/vdb
  Disk /dev/vdb: 50 GiB, 53687091200 bytes, 104857600 sectors
  Units: sectors of 1 * 512 = 512 bytes
  Sector size (logical/physical): 512 bytes / 512 bytes
  I/O size (minimum/optimal): 512 bytes / 512 bytes

Now use ``fdisk`` to create a partition on the disk:

.. code-block:: bash

  $ sudo fdisk /dev/vdb

  Welcome to fdisk (util-linux 2.27.1).
  Changes will remain in memory only, until you decide to write them.
  Be careful before using the write command.

  Device does not contain a recognized partition table.
  Created a new DOS disklabel with disk identifier 0x1552cd32.

  Command (m for help): n
  Partition type
     p   primary (0 primary, 0 extended, 4 free)
     e   extended (container for logical partitions)
  Select (default p): p
  Partition number (1-4, default 1): 1
  First sector (2048-104857599, default 2048):
  Last sector, +sectors or +size{K,M,G,T,P} (2048-104857599, default 104857599):

  Created a new partition 1 of type 'Linux' and of size 50 GiB.

  Command (m for help): w
  The partition table has been altered.
  Calling ioctl() to re-read partition table.
  Syncing disks.

Check the partition using ``lsblk``:

.. code-block:: bash

  NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
  vda    253:0    0  10G  0 disk
  └─vda1 253:1    0  10G  0 part /
  vdb    253:16   0  50G  0 disk
  └─vdb1 253:17   0  50G  0 part

Make a new filesystem on the partition:

.. code-block:: bash

  $ sudo mkfs.ext4 /dev/vdb1
  mke2fs 1.42.13 (17-May-2015)
  Creating filesystem with 5242624 4k blocks and 1310720 inodes
  Filesystem UUID: 7dec7fb6-ff38-453b-9335-0c240d179262
  Superblock backups stored on blocks:
      32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
      4096000

  Allocating group tables: done
  Writing inode tables: done
  Creating journal (32768 blocks): done
  Writing superblocks and filesystem accounting information: done

Create a directory where you wish to mount this file system:

.. code-block:: bash

  $ sudo mkdir /mnt/extra-disk

Mount the file system:

.. code-block:: bash

  $ sudo mount /dev/vdb1 /mnt/extra-disk

Label the partition:

.. code-block:: bash

  $ sudo tune2fs -L 'extra-disk' /dev/vdb1
  tune2fs 1.42.13 (17-May-2015)
  $ sudo blkid
  /dev/vda1: LABEL="cloudimg-rootfs" UUID="98c51306-83a2-49da-94a9-2a841c9f27b0" TYPE="ext4" PARTUUID="8cefe526-01"
  /dev/vdb1: LABEL="extra-disk" UUID="7dec7fb6-ff38-453b-9335-0c240d179262" TYPE="ext4" PARTUUID="235ac0e4-01"

If you want the new file system to be mounted when the system reboots then you
should add an entry to ``/etc/fstab``, for example:

.. code-block:: bash

  $ cat /etc/fstab
  LABEL=cloudimg-rootfs /               ext4    defaults    0 1
  LABEL=extra-disk      /mnt/extra-disk ext4    defaults    0 2

.. note::

  When referring to block devices in ``/etc/fstab`` it is recommended that UUID
  or volume label is used instead of using the device name explicitly. It is
  possible for device names to change after a reboot particularity when there are
  multiple attached volumes.

***
FAQ
***

How to grow a cinder volume?
============================

So you have been succesfully using OpenStack and now one of your volumes has
started filling up.  What is the best, quickest and safest way to grow the
size of your volume?

Well, as always, that depends.

Boot Volumes
============

This is difficult in OpenStack as there is not an easy and obvious choice.

Create New Instance
-------------------

The best method is to spin up a new instance with a new volume and use
the configuration management tool of your choice to make sure it is as you
want it.  Terminate the old instance and attach all the data volumes to the
new instance.

This assumes there is no permanent data stored on the boot volume that is
outside the configuration managment tool control.

Use a Volume Snapshot
---------------------

Another method which is quick and safe is to perform a volume snapshot.

The process is as follows:

* Shutdown the instance.
* Take a volume snapshot.
* Create volume from snapshot.
* Boot instance from volume.

This sequence can be performed either through the API/commands or the
dashboard.

A reason to like this method is that the original volume is maintained, it is
quick and cloud-init grows the new instance filesystem to the new volume size
on first boot.

The reasons not to like this method are:

* The host gets new keys which may upset some applications.
* The original volume and the snapshot can not be deleted until the newly
  created volume is deleted.
* You will be charged for these cinder volumes and the snapshot.

Old Fashioned Method
--------------------

Finally, there is the old fashioned methods that involves:

* Create a new bootable volume.
* Shutdown instance and detach boot volume.
* Attach the new volume and the original to another instance.
* Perform a copy using tools like dd.

Non-boot Volumes
================

The way to go is:

* Detach the volume from the instance
* Extend the volume
* Attach the volume to the instance
* Adjust the disk within the OS as you would normally
