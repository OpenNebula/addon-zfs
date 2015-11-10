Opennebula ZFS Storage Driver
=============================

Use it in single host, for multiple hosts you may use [opennebula-addon-iscsi](https://github.com/kvaps/opennebula-addon-iscsi) (with zfs support)

## Installation

  - copy `datastore/zfs` to `/var/lib/one/datastore/zfs`
  - copy `tm/zfs` to `/var/lib/one/tm/zfs`
  - edit `/etc/one/oned.conf`:

```bash
# Change TM_MAD arguments to add zfs:
TM_MAD = [
    executable = "one_tm",
    arguments = "-t 15 -d dummy,lvm,shared,fs_lvm,qcow2,ssh,vmfs,ceph,dev,zfs"
]

# Change DATASTORE_MAD arguments to add zfs:
DATASTORE_MAD = [
    executable = "one_datastore",
    arguments  = "-t 15 -d dummy,fs,vmfs,lvm,ceph,dev,zfs"
]

# Add this to the end of file: 
TM_MAD_CONF = [
    name = "zfs", ln_target = "NONE", clone_target = "SELF", shared = "yes"
]

```
  - add `oneadmin` user to `disk` group

  - run `visudo` then add:
```bash
%oneadmin    ALL=(root) NOPASSWD: /usr/sbin/zfs
```

  - create datastore:

```bash
cat << EOT > ds.conf
NAME = "zfs"
DS_MAD = zfs
TM_MAD = zfs
DISK_TYPE = block
DATASET_NAME = rpool/ONE/images
BRIDGE_LIST ="localhost"
EOT

onedatastore create ds.conf
```
