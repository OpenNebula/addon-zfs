# Opennebula ZFS Storage Driver 

## Description

The ZFS datastore driver provides OpenNebula with the possibility of using ZVOL volumes instead of plain files to hold the Virtual Images.

## Development

To contribute bug patches or new features, you can use the github Pull Request model. It is assumed that code and documentation are contributed under the Apache License 2.0. 

More info:
* [How to Contribute](http://opennebula.org/addons/contribute/)
* Support: [OpenNebula user forum](https://forum.opennebula.org/c/support)
* Development: [OpenNebula developers forum](https://forum.opennebula.org/c/development)
* Issues Tracking: Github issues (https://github.com/OpenNebula/addon-zfs/issues)

## Author

* [kvaps](mailto:kvapss@gmail.com)

## Compatibility

This add-on is compatible with OpenNebula 4.6+

## Requirements

### OpenNebula Front-end

Password-less ssh access to an OpenNebula ZFS host. (eg. localhost)

### OpenNebula ZFS Host

The oneadmin user should be able to execute ZFS related command with sudo passwordlessly.

* Password-less sudo permission for: `zfs` and `dd`.
* `oneadmin` needs to belong to the `disk` group (for KVM).

## Limitations

There are some limitations that you have to consider, though:

* ZFS is not cluster filesystem! Use it on single host. For multiple hosts you may use [ceph](http://docs.opennebula.org/4.10/administration/storage/ceph_ds.html) or [opennebula-addon-iscsi](https://github.com/kvaps/opennebula-addon-iscsi) with zfs support.

## Installation

To install the driver you have to copy these files:

* `datastore/zfs` --> `/var/lib/one/datastore/zfs`
* `tm/zfs` --> `/var/lib/one/tm/zfs`

## Configuration

### Configuring the System Datastore

To use ZFS drivers, you must configure the system datastore as shared. This system datastore will hold only the symbolic links to the block devices, so it will not take much space. See more details on the [System Datastore Guide](http://docs.opennebula.org/4.6/administration/storage/system_ds.html).

It will also be used to hold context images and Disks created on the fly, they will be created as regular files.

### Configuring ZFS Datastores

The first step to create a ZFS datastore is to set up a template file for it. In the following table you can see the supported configuration attributes. The datastore type is set by its drivers, in this case be sure to add `DS_MAD=zfs` and `TM_MAD=zfs` for the transfer mechanism, see below.

|    Attribute        |                     Description                                                                                                                                      |
| ---------------     | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `NAME`              | The name of the datastore                                                                                                                                            |
| `DS_MAD`            | Must be `zfs`                                                                                                                                                        |
| `TM_MAD`            | Must be `zfs`                                                                                                                                                        |
| `DISK_TYPE`         | Must be `block`                                                                                                                                                      |
| `BRIDGE_LIST`       | The zfs server host. Defaults to `localhost`                                                                                                                         |
| `DATASET_NAME`      | The top level dataset name under which all volumes are created. Defaults to `rpool/ONE/images`                                                                       |
| `ZFS_CMD`           | Path to the zfs binary. Defaults to `/usr/sbin/zfs`                                                                                                                  |
| `RESTRICTED_DIRS`   | Paths that can not be used to register images. A space separated list of paths. (1)                                                                                  |
| `SAFE_DIRS`         | If you need to un-block a directory under one of the RESTRICTED_DIRS. A space separated list of paths.                                                               |
| `NO_DECOMPRESS`     | Do not try to untar or decompress the file to be registered. Useful for specialized Transfer Managers                                                                |
| `LIMIT_TRANSFER_BW` | Specify the maximum transfer rate in bytes/second when downloading images from a http/https URL. Suffixes K, M or G can be used.                                     |


> (1) This will prevent users registering important files as VM images and accessing them through their VMs. OpenNebula will automatically add its configuration directories: /var/lib/one, /etc/one and oneadmin's home. If users try to register an image from a restricted directory, they will get the following error message: “Not allowed to copy image file”.

For example, the following examples illustrates the creation of an ZFS datastore using a configuration file. In this case we will use the host `localhost` as ZFS-enabled host.

~~~~
> cat ds.conf
NAME = "zfs"
DS_MAD = zfs
TM_MAD = zfs
DISK_TYPE = block
DATASET_NAME = rpool/ONE/images

> onedatastore create ds.conf
ID: 100

> onedatastore list
  ID NAME            CLUSTER  IMAGES TYPE   TM    
   0 system          none     0      fs     shared
   1 default         none     3      fs     shared
 100 zfs             none     0      zfs    shared
~~~~

The DS and TM MAD can be changed later using the onedatastore update command. You can check more details of the datastore by issuing the onedatastore show command.

> Note that datastores are not associated to any cluster by default, and they are supposed to be accessible by every single host. If you need to configure datastores for just a subset of the hosts take a look to the [Cluster guide](http://opennebula.org/documentation:rel4.4:cluster_guide).

### Configuring DS_MAD and TM_MAD

These values must be added to `/etc/one/oned.conf`

First we add `zfs` as an option, replace:

~~~~
TM_MAD = [
    executable = "one_tm",
    arguments = "-t 15 -d dummy,lvm,shared,fs_lvm,qcow2,ssh,vmfs,ceph"
]
~~~~

With:

~~~~
TM_MAD = [
    executable = "one_tm",
    arguments = "-t 15 -d dummy,lvm,shared,fs_lvm,qcow2,ssh,vmfs,ceph,zfs"
]
~~~~

After that create a new TM_MAD_CONF section:

~~~~
TM_MAD_CONF = [
    name = "zfs", ln_target = "NONE", clone_target = "SELF", shared = "yes"
]
~~~~

Now we add `zfs` as a new `DATASTORE_MAD` option, replace:

~~~~
DATASTORE_MAD = [
    executable = "one_datastore",
    arguments  = "-t 15 -d dummy,fs,vmfs,lvm,ceph"
]
~~~~

With:

~~~~
DATASTORE_MAD = [
    executable = "one_datastore",
    arguments  = "-t 15 -d dummy,fs,vmfs,lvm,ceph,zfs"
]
~~~~

## Usage 

The ZFS transfer driver will create volume with zfs. Once the zvol is available, the driver will link it to `disk.i`.

### Host Configuration

The host must have ZFS and have the dataset used in the `DATASET` attributed of the datastore template. 

It’s also required to have password-less sudo permission for `zfs` and `dd`.

## Tuning & Extending

System administrators and integrators are encouraged to modify these drivers in order to integrate them with their datacenter:

Under ``/var/lib/one/remotes/``:

-  **datastore/zfs/zfs.conf**: Default values for LVM parameters

   -  ZFS_CMD: Path to the zfs binary
   -  BRIDGE_LIST: The zfs server host
   -  DATASET_NAME: Default dataset
   -  STAGING_DIR: Staging directory

-  **datastore/zfs/cp**: Registers a new image. Creates a new ZFS volume.
-  **datastore/zfs/mkfs**: Makes a new empty image. Creates a new ZFS volume.
-  **datastore/zfs/rm**: Removes the ZFS volume.
-  **tm/zfs/ln**: Links to the ZFS volume.
-  **tm/zfs/clone**: Clones the image by creating a snapshot.
-  **tm/zfs/mvds**: Saves the image in a new ZFS volume for SAVE\_AS.

## Optimizations

* Due [this issue](https://github.com/zfsonlinux/zfs/issues/824) use more large values of `volblocksize`.
* Also you may to turn on the writeback cache or set `io` to `native`


