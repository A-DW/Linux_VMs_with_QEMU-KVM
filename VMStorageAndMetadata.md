# VM storage and metadata

This reference covers Ubuntu hosts using `qemu:///system`. Run filesystem commands on the virtualization host, not the remote virt-manager workstation. System and session connections have different inventories. Paths below are common defaults, not proof of a particular VM's configuration; packages, versions and local settings can change them.

## Inventory before backup or deletion

```bash
URI=qemu:///system
VM=lab-ubuntu-01
virsh -c "$URI" hostname
virsh -c "$URI" uri
virsh -c "$URI" dominfo "$VM"
virsh -c "$URI" domuuid "$VM"
virsh -c "$URI" dumpxml "$VM" --inactive
virsh -c "$URI" domblklist "$VM" --inactive --details
virsh -c "$URI" domiflist "$VM" --inactive
virsh -c "$URI" snapshot-list "$VM"
virsh -c "$URI" checkpoint-list "$VM"
```

Record the host, URI, name, UUID, disk targets, exact source paths, firmware state, TPM configuration and secret references. Check command errors: an unsupported command or inaccessible service is not an empty inventory. For an active VM also inspect live XML and disk mappings; reconcile differences before maintenance.

## Component locations

| Component | Common system-instance path | Discovery and ownership |
| --- | --- | --- |
| Persistent definition | `/etc/libvirt/qemu/<name>.xml` | Export with `dumpxml --inactive`; manage through libvirt, not manual XML deletion |
| Autostart entry | `/etc/libvirt/qemu/autostart/<name>.xml` | Usually a symlink; inspect `dominfo` |
| Guest disks | `/var/lib/libvirt/images/` | Actual sources come from `domblklist` and XML; custom paths, block devices and remote storage are possible |
| UEFI NVRAM | `/var/lib/libvirt/qemu/nvram/` | Inspect the XML `nvram` configuration; preserve the writable state, not just the shared firmware template |
| Other UEFI variable storage | Version-dependent, potentially `/var/lib/libvirt/qemu/varstore/` | Inspect XML `varstore` where supported; do not assume NVRAM is the only firmware state |
| Emulated TPM persistent state | `/var/lib/libvirt/swtpm/<UUID>/` | Confirm local configuration; usually UUID-based, not name-based; external TPM backends differ |
| Managed-save image | `/var/lib/libvirt/qemu/save/<name>.save` | Check `dominfo`; this is saved execution state, not a disk backup |
| Snapshot metadata | `/var/lib/libvirt/qemu/snapshot/<name>/` | Enumerate with `snapshot-list` and export each needed record using `snapshot-dumpxml` |
| Checkpoint metadata | `/var/lib/libvirt/qemu/checkpoint/<name>/` | Enumerate with `checkpoint-list` and export needed records with `checkpoint-dumpxml` |
| External snapshots and backing files | User-selected paths | Inspect snapshot XML and disk chains; filenames may not contain the VM name |
| Memory saves and dumps | User-selected paths; automatic dumps may use `/var/lib/libvirt/qemu/dump/` | Record exact paths; memory can contain credentials |
| Runtime state and sockets | `/run/libvirt/qemu/`, with working files under `/var/lib/libvirt/qemu/` | Not ordinary cold-backup inputs; let libvirt manage runtime cleanup |
| QEMU logs | `/var/log/libvirt/qemu/<name>.log*` | Historical records can remain after deletion |
| TPM logs | `/var/log/swtpm/libvirt/qemu/` | Distinct from persistent TPM state |
| Libvirt secrets | `/etc/libvirt/secrets/` | Inspect secret references by UUID; secrets may be shared and require separate secure recovery procedures |
| Network definitions | `/etc/libvirt/qemu/networks/` | Independent resources, often shared |
| Storage pool definitions | `/etc/libvirt/storage/` | Pools are not individual VM disks |
| ISO images | This repository uses `/var/lib/libvirt/boot/` | Installation media are normally shared |
| Backups and exported inventories | Operator-selected destination | Protect separately; these remain recoverable copies |

The table is a discovery aid, not a deletion list. Validate paths on your host before using them. Do not copy or remove whole libvirt trees as a per-VM procedure.

For a stopped file-backed guest, inspect each disk's backing chain:

```bash
sudo qemu-img info --backing-chain --output=json /exact/path/disk.qcow2
```

Do not modify an image in use. Chain inspection does not discover all historical snapshots, replicas or backups. Also inspect external data-file references and storage-platform snapshots. Mark each resource as exclusive, shared or unknown. Unknown ownership blocks deletion.

## Networks and shared resources

The example `--network network=default,model=virtio` attaches a NIC to an existing network. It does not create a dedicated network. Keep `default`, its bridge, shared pools, installation media, firmware templates and shared backing images when deleting one VM. Review custom DHCP reservations separately. A host bridge may be managed by Netplan rather than libvirt.

Continue with [VM lifecycle management](VMLifecycleManagement.md).

## Authoritative references

- [Libvirt QEMU driver](https://libvirt.org/drvqemu.html)
- [Domain XML](https://libvirt.org/formatdomain.html)
- [virsh command documentation](https://libvirt.org/manpages/virsh.html)
- [QEMU image tools](https://www.qemu.org/docs/master/tools/qemu-img.html)
