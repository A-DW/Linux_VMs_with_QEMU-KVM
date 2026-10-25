# VM backup, restoration, deletion and verification

Scope: Ubuntu, libvirt and `qemu:///system`. Run commands on the host containing the VM. These are operator-reviewed procedures, not an automatic deletion script. Read [VM storage and metadata](VMStorageAndMetadata.md) and record an exact resource inventory first.

## Choose the intended operation

| Operation | Result |
| --- | --- |
| `shutdown` | Requests a graceful guest shutdown; asynchronous |
| `destroy` | Abrupt power-off, not deletion; risks guest data loss |
| `undefine` | Removes persistent registration; running guests can remain transient |
| Delete selected storage/state | Removes explicitly selected resources, subject to command support and errors |
| Sanitize | Addresses recoverability at the encryption/storage level; not established by an absent pathname |

## 1. Inventory and protect recovery information

Record name, UUID, host, URI, exact disk and state paths, disk dependencies, network sources and secret references before undefining. Never rely only on name searches. Store inventories and backups outside the public repository, on protected storage, with a documented retention policy.

```bash
URI=qemu:///system
VM=lab-ubuntu-01
virsh -c "$URI" domuuid "$VM"
virsh -c "$URI" dumpxml "$VM" --inactive
virsh -c "$URI" domblklist "$VM" --inactive --details
virsh -c "$URI" help undefine
```

## 2. Offline backup

The beginner workflow is a cold backup. Do not copy active QCOW2 images with ordinary file-copy commands. A snapshot alone is not an independent backup.

1. Arrange downtime and stop applications cleanly in the guest.
2. Run `virsh -c "$URI" shutdown "$VM"`, then poll `virsh -c "$URI" domstate "$VM"` until it reports `shut off`. A suspended guest is not shut off. Prevent automatic or concurrent restarts while copying. Resolve any managed-save state intentionally; do not mix old saved memory with unrelated disk state.
3. Reconcile the final disk inventory and ensure no backup/block job or other writer is using the disks.
4. Create a restricted directory on an existing, verified backup filesystem. Do not assume an absent mount means the destination disk is available.
5. Export the inactive XML and record name/UUID and dependency inventory.
6. Copy each required disk and its dependencies. Copy matching writable firmware variables and emulated TPM persistent state while stopped. Keep these components from the same backup point. Protect encryption keys and recovery material separately.
7. Export required snapshot/checkpoint metadata individually if preserving that history. Record firmware packages, CPU/machine requirements, network configuration and secret recovery dependencies.
8. Generate and verify checksums, then test restoration from the backup copies.

Example for an already verified mounted backup destination:

```bash
BACKUP=/mnt/vm-backups/lab-ubuntu-01-cold-backup
umask 077
mkdir -m 700 "$BACKUP"
virsh -c "$URI" dumpxml "$VM" --inactive > "$BACKUP/domain.xml"
virsh -c "$URI" domuuid "$VM" > "$BACKUP/uuid.txt"
```

Stop if any command fails. Do not overwrite an existing backup directory. For a simple standalone disk, copy its exact inventoried source to a unique destination, for example:

```bash
sudo cp --sparse=always --preserve=all -- \
  /var/lib/libvirt/images/lab-ubuntu-01.qcow2 "$BACKUP/vda.qcow2"
sudo sha256sum /var/lib/libvirt/images/lab-ubuntu-01.qcow2 "$BACKUP/vda.qcow2"
```

Both digests should match. Repeat for every required disk and state file; copying this one image is not a complete UEFI/TPM backup. Retain a mapping of original paths to backup paths. Root-owned copies may require sudo to inspect. Restrict access to copied files as well as their containing directory.

For backing chains, either preserve the complete required chain and valid path relationships, or deliberately create a standalone representation of the current disk view with `qemu-img convert`. A converted current view does not preserve all snapshot history. Block/network storage, encrypted TPM state and passthrough devices require backend-specific recovery procedures; the simple file-copy example is insufficient.

## 3. Restore and prove the backup works

Prefer an isolated test host. Keep the restored guest disconnected from the production network initially.

1. Verify stored checksums. For supported image formats, run read-only `qemu-img check` on offline backup copies; never use repair options as routine verification.
2. Restore disks, matching firmware variables and TPM state to reviewed paths. Do not point the test VM at original live disks.
3. Make a working copy of the exported XML and update disk, firmware and other host-specific paths. Recreate required networks and securely restore secret dependencies. Set ownership, permissions and security labels appropriate for the destination host; do not solve permission failures with world-writable permissions.
4. On an isolated destination with no conflicting VM identity, register the reviewed definition using `virsh -c qemu:///system define /exact/path/restored-domain.xml`.
5. Start the guest using `virsh -c qemu:///system start VM_NAME`. Test boot, important files, applications, networking when safely enabled, and TPM/encryption-dependent functions.
6. Record results and confirm the original VM was unaffected.

Same-host duplicate restoration needs deliberate name, UUID, MAC address and TPM identity handling; do not blindly reuse the original identity. Checksums and structural checks do not replace a functional restore test.

## 4. Unregister while retaining data

Export and verify recovery information first. With the guest shut off, a VM without additional state may be unregistered using `virsh -c "$URI" undefine "$VM"`. Disks are retained when no storage-removal option is supplied.

For UEFI or TPM guests, inspect the installed `help undefine` and explicitly select supported `--keep-nvram` and `--keep-tpm` options where applicable. Managed-save, snapshot and checkpoint metadata can block undefine; decide whether to preserve/export or remove them before proceeding. Do not add removal flags merely to silence an error. Re-registering XML alone cannot restore state that was deleted.

## 5. Delete selected VM-owned resources

Destructive: first verify the backup decision, exact host/URI/name/UUID, shut-off state and exclusive ownership of every target. Check other powered-off guests and external consumers, not only running VMs. Stop on unknown ownership.

For a reviewed example with only the managed disk target `vda` selected for deletion:

```bash
virsh -c "$URI" undefine "$VM" --storage vda
```

This is not a universal command. Build the final command from the installed help and the inventory:

| Option | Intended effect |
| --- | --- |
| `--storage vda,vdb` | Delete only selected attached libvirt-managed volumes |
| `--remove-all-storage` | Broader alternative; use only when every selected attached volume is expendable and exclusive |
| `--nvram` | Delete firmware NVRAM rather than keep it |
| `--tpm` | Explicitly delete emulated TPM persistent state where supported |
| `--managed-save` | Remove managed-save state |
| `--snapshots-metadata` | Remove libvirt snapshot metadata, not every external snapshot disk |
| `--checkpoints-metadata` | Remove checkpoint metadata; coordinate with incremental backup management |

### Condensed deletion command

After completing the inventory and backup decision, confirming the VM is shut off, and checking that every attached managed disk is exclusively owned and expendable, the following combines the common deletion options. Run it on the virtualization host with `VM` set to the exact VM name, and first confirm that the installed `virsh help undefine` supports these options.

```bash
virsh -c qemu:///system undefine "$VM" \
  --remove-all-storage \
  --nvram \
  --managed-save \
  --snapshots-metadata \
  --checkpoints-metadata
```

This is destructive, not a secure-erasure guarantee. It does not explicitly request TPM-state deletion; for an emulated TPM, review the inventory and add `--tpm` only if supported and that state is intended for deletion. External snapshot disks, backing files, unmanaged storage, backups and other copies require separate review. Check all command output and complete the verification procedure below, even if the command reports success.

Do not combine keep and delete options for the same state. Flag support and firmware handling vary by version. Only managed volumes can be removed this way; only the top image of a backing chain is removed. Read every message: registration can be removed while volume deletion fails. A source not found in the definition can be treated as already removed, so command success is not sufficient evidence.

Review remaining unmanaged disks, unused external snapshots, memory saves/dumps, exported XML, backups and replicas by exact inventory entry. Use a backend-appropriate operation only after confirming ownership. Never use wildcard deletion or recursively remove libvirt directories. Preserve shared networks, pools, firmware, ISOs, base images and shared secrets. Logs may remain under an intentional retention policy.

## 6. Verify deletion

Record PASS, FAIL or UNKNOWN for each check. Connection errors, permission failures and unavailable storage are UNKNOWN, never proof of absence.

```bash
virsh -c "$URI" hostname
virsh -c "$URI" list --all --name
virsh -c "$URI" list --all --uuid
```

Successful enumeration must show neither the old name nor UUID. Include active domains: undefining a running guest can leave it transient. Check original storage is mounted/reachable before testing exact paths. For each inventoried resource, inspect its parent and exact path with appropriate privileges. Confirm expected absence, not merely a failed command. Refresh and inspect relevant storage pools; do not assume all disks belong to `default`.

Secondary search on the host, replacing both identifiers:

```bash
VM=lab-ubuntu-01
VM_UUID=REPLACE_WITH_RECORDED_UUID
sudo find /etc/libvirt /var/lib/libvirt /run/libvirt \
  \( -iname "*${VM}*" -o -iname "*${VM_UUID}*" \) -print
```

This searches filenames only in the listed locations. It does not prove absence of custom paths, remote storage, differently named copies or content references. Review logs separately under `/var/log/libvirt` and `/var/log/swtpm`; their presence does not mean the VM is registered.

| Check | Required evidence |
| --- | --- |
| Registration/runtime | Successful enumeration on the correct host contains neither name nor UUID |
| Selected storage/state | Each inventoried object is absent on reachable original storage |
| Metadata | Inventoried definition, autostart, save, snapshot/checkpoint and firmware/TPM state have the intended disposition |
| Other copies | Explicit disposition for snapshots, backups, replicas and exports |
| Shared resources | Other guests and required shared networking/storage still work |

Report the scope accurately, for example: 'VM registration and inventoried local resources deleted; backup intentionally retained; sanitization not established.'

## 7. Secure erasure is a separate requirement

Deleting files, `--wipe-storage`, `shred`, overwriting guest free space or issuing discard does not universally establish irrecoverability. SSD wear levelling, remapped blocks, copy-on-write filesystems, snapshots, caches and backups can preserve copies.

Choose a storage-specific sanitization method and document its scope and evidence. Per-VM cryptographic erasure must be designed with independent encryption keys and complete accounting for recovery material, key copies, TPM state and backups. Deleting one libvirt secret is not proof that all decryption capability is gone. Whole-device sanitization destroys other data on that device and is not a routine per-VM command. No whole-device destructive recipe is provided here.

## Validation status

These procedures are documentation, not evidence of tests on your host. Before relying on them, test disposable BIOS, UEFI and UEFI-plus-TPM guests, custom storage, backing chains, shared resources and failure cases. Record package versions and results. Online/incremental backup automation is outside this beginner guide.

## References

- [virsh commands](https://libvirt.org/manpages/virsh.html)
- [Domain XML](https://libvirt.org/formatdomain.html)
- [Libvirt backup architecture](https://libvirt.org/kbase/live_full_disk_backup.html)
- [QEMU image tools](https://www.qemu.org/docs/master/tools/qemu-img.html)
- [GNU shred limitations](https://www.gnu.org/software/coreutils/manual/html_node/shred-invocation.html)
