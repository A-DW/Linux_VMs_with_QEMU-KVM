# Unanswered Questions

In this document I go through some questions and doubts that I had.

## Questions:

- [Why both?](#why-both)
- [What is `qemu:///system`?](#what-is-qemusystem)
- [System vs session](#system-vs-session)
- [Are they separate environments?](#are-they-separate-environments)
- [Can the URI have different words?](#can-the-uri-have-different-words)
- [Why not `kvm:///system`?](#why-not-kvmsystem)
- [How do shutdown, destroy, undefine and deletion differ?](#how-do-shutdown-destroy-undefine-and-deletion-differ)
- [Is a snapshot the same as a backup?](#is-a-snapshot-the-same-as-a-backup)
- [Are the VM definition and virtual disk the whole machine?](#are-the-vm-definition-and-virtual-disk-the-whole-machine)
- [Should deleting a VM also delete its network?](#should-deleting-a-vm-also-delete-its-network)
- [Does a missing VM entry prove complete deletion?](#does-a-missing-vm-entry-prove-complete-deletion)
- [Is deletion the same as secure erasure?](#is-deletion-the-same-as-secure-erasure)
- [Sources](#sources)

**QEMU and KVM serve different purposes**, but on Linux they are normally used **together**. QEMU provides the virtual computer; KVM lets QEMU execute the guest's CPU instructions efficiently on the physical CPU.

## Why both?

Think of a VM as needing two major components:

| Component | Responsibility |
| --- | --- |
| QEMU | Construct the virtual computer: motherboard, RAM, disks, NICs, USB controllers, firmware, display adapters and other devices |
| KVM | Provides hardware-accelerated execution of the guest CPU through the Linux kernel |
| libvirt | Starts, stops and configures QEMU processes |
| virt-manager | Graphical interface controlling libvirt |

The relationship is approximately:
```text
virt-manager
     ↓
libvirt
     ↓
QEMU ─────→ virtual disks, NICs, display, firmware, devices
     │
     └─────→ KVM → accelerated guest CPU execution
```

QEMU can run without KVM by using its software CPU translator, **TCG**. That enables full emulation — including running software built for a different CPU architecture — but it is generally much slower than KVM for ordinary same-architecture VMs.

KVM, conversely, does not provide the complete user-facing virtual computer by itself. It exposes a Linux kernel API, normally through ``/dev/kvm``, which a userspace program such as QEMU uses to execute the VM's virtual CPUs.

Therefore:
```text
QEMU without KVM = complete emulation, but usually slower
KVM without QEMU = acceleration API, but no practical complete VM environment
QEMU with KVM = complete VM with hardware-accelerated execution
```

For example, QEMU could run an ARM guest on an x86 computer through emulation:
```bash
qemu-system-aarch64 ...
```

KVM cannot accelerate that combination because the guest and host instruction architectures differ. But for an x86-64 guest on an x86-64 Ubuntu host, QEMU can use:
```bash
qemu-system-x86_64 -accel kvm ...
```

This is why people conventionally call the stack **QEMU/KVM**.

## What is ``qemu:///system``?

``qemu:///system`` is a **libvirt connection URI**. It does not start QEMU directly and is not a filesystem path.

It tells applications such as ``virsh`` and virt-manager:
> Connect to the local system-wide libvirt QEMU driver.

Its parts are:
```text
qemu:///system
│       │
│       └── libvirt system instance
│
└── QEMU/libvirt driver
```

A more precise URI breakdown is:
```text
qemu:// /system
│       │
│       └── path
│
└── URI scheme
```

There are **three slashes** because no remote hostname appears between **//** and **/system**:
```text
qemu://hostname/system     Remote hostname present
qemu:///system             Hostname omitted; local machine
```

The word ``qemu`` selects libvirt's QEMU driver. That driver can manage both software-emulated QEMU machines and KVM-accelerated QEMU machines, which is why the URI is named **qemu**, not necessarily **kvm**.

## System vs session

The two principal local URIs are:

| URI | Meaning |
| --- | --- |
| ``qemu:///system`` | Connect to the system-wide libvirt instance |
| ``qemu:///session`` | Connect to your personal, unprivileged libvirt instance |

### ``qemu:///system``

```bash
virsh -c qemu:///system list --all
```

This connects to the system-level virtualization environment. It is suitable for:
 - VMs that start when the host boots
 - Libvirt-managed NAT networks
 - Linux bridges
 - PCI, USB and other host-device access
 - Server and home-lab VMs
 - VMs shared between authorized administrators

The management service is privileged, although QEMU guest processes are normally subjected to additional user, permission and security confinement configured by the distribution. Access to the system libvirt API should be treated as highly privileged.

### ``qemu:///session``

```bash
virsh -c qemu:///session list --all
```

This connects to a libvirt instance associated with your user account. Its VMs run with your user's permissions and commonly store their disk images under your home directory. It has fewer permission complications but more restrictions around networking, host devices and boot-time operation.

Most of your home-lab work should use:
```text
qemu:///system
```

## Are they separate environments?

Yes. This is very important:
```bash
virsh -c qemu:///system list --all
virsh -c qemu:///session list --all
```

These commands may show **completely different VM lists**.

A VM created under ``qemu:///system`` does not automatically appear under ``qemu:///session``, even though both connections are on the same computer. They represent separate libvirt management scopes with separate VM definitions, storage configuration and networking capabilities.

This sometimes causes the confusing situation where virt-manager shows a VM but ``virsh list --all`` does not. Usually, virt-manager and ``virsh`` are connected to different URIs.

Always make the connection explicit while learning:
```bash
virsh --connect qemu:///system list --all
```
The shorter equivalent is:
```bash
virsh -c qemu:///system list --all
```

## Can the URI have different words?

Yes. Different words change the driver, privilege scope, transport or destination.

### Local session instance

```text
qemu:///session
```

Same QEMU driver, but your per-user libvirt environment.

### Remote host over SSH

```text
qemu+ssh://user@server/system
```
This means:
```text
qemu       → Use the QEMU driver
+ssh       → Transport the libvirt connection over SSH
server     → Remote virtualization host
/system    → Use that host’s system libvirt instance
```

Example:
```bash
virsh -c qemu+ssh://alex@192.168.1.50/system list --all
```

Virt-manager can use this too, allowing your Ubuntu desktop to manage VMs running on a separate Linux server. libvirt officially supports local, Unix socket, SSH, TCP and remote variants of the QEMU connection URI.

### Explicit local Unix transport
```text
qemu+unix:///system
```
This explicitly requests a local Unix socket. For normal local operation, it is usually unnecessary because:
```text
qemu:///system
```
already resolves to the appropriate local connection.

### Other virtualization drivers

Libvirt supports more than QEMU, so the first word can theoretically select another driver:
```text
lxc:///system
xen:///system
```

Those do not mean "alternative names for the QEMU/KVM connection"; they identify different virtualization backends. Libvirt uses connection URIs precisely because it can manage multiple virtualization providers.

## Why not ``kvm:///system``?

The documented libvirt URI scheme for this driver is ``qemu``, because QEMU is the userspace VM monitor being managed. Whether a particular guest uses KVM acceleration is stored in that VM's definition rather than determined by the URI.

You can inspect a VM's definition:
```bash
virsh -c qemu:///system dumpxml VM_NAME | head
```

A KVM-accelerated guest will typically begin with something resembling:
```xml
<domain type='kvm'>
```
A software-emulated QEMU guest would instead use:
```xml
<domain type='qemu'>
```
Therefore, these two concepts answer different questions:
```text
qemu:///system
```
means:
> Which libvirt driver and management environment am I connecting to?

Whereas:
```xml
<domain type='kvm'>
```
means:
> Which execution backend should this particular VM use?

So the precise recommendation is:
> Use **QEMU as the virtual-machine monitor, KVM as QEMU's hardware accelerator, libvirt as the management layer, and connect virt-manager/virsh to ``qemu:///system``**.

## How do shutdown, destroy, undefine and deletion differ?

A VM has running state, a persistent libvirt definition, disks and other state. Stopping one component does not automatically remove the others.

| Operation | What changes | What it does not establish |
| --- | --- | --- |
| `shutdown` | Requests a graceful guest shutdown; wait for `shut off` | Does not delete registration, disks or metadata |
| `destroy` | Forces power-off, similar to cutting power | Does not mean delete; may lose unsaved data |
| `undefine` without storage-removal options | Removes persistent registration; extra state can require explicit keep/remove choices | Does not delete disks; if running, the guest can continue as a transient domain |
| Delete selected resources | Removes reviewed disks/state using explicit supported options or backend-specific operations | Does not automatically find all snapshots, backups or other copies |
| Sanitize | Applies a storage/encryption-specific method intended to prevent recovery | Is not proven by a successful undefine or missing filename |

Shut the VM off before following this repository's deletion workflow. Inspect `virsh help undefine` on the installed version and make explicit decisions about firmware variables, TPM state, managed saves and snapshot/checkpoint metadata. See [VM lifecycle management](VMLifecycleManagement.md) for procedures, including the condensed deletion command.

## Is a snapshot the same as a backup?

No. A snapshot records a point in time and can depend on the original disk or backing chain. It may also reside on the same storage as the VM. Losing that storage can therefore lose both the VM and its snapshots.

A backup must contain the data and dependencies needed for the intended recovery, with appropriately protected independent copies. For UEFI/TPM guests, disks alone may be insufficient: matching writable firmware variables, TPM state and required key/recovery material can matter. An XML export describes configuration but does not contain the guest filesystem.

Checksums verify copied content; supported image checks inspect structural integrity. Neither proves that the guest and its applications can be restored. Complete an isolated functional restore test. The [lifecycle guide](VMLifecycleManagement.md#2-offline-backup) starts with an offline backup rather than copying active disk images.

## Are the VM definition and virtual disk the whole machine?

Not necessarily. Separate components can include writable UEFI variables, emulated TPM state, managed-save memory, snapshot/checkpoint metadata, external snapshot disks, backing files and referenced secrets. Logs and exported backups are additional artifacts, but are not the running VM itself.

Snapshot metadata describes snapshots; deleting that metadata is not the same as deleting every external snapshot disk. Saved memory is also different from a disk backup and can contain sensitive information.

There is no universal per-VM directory. Common system-instance locations include `/etc/libvirt/qemu/` for definitions, `/var/lib/libvirt/images/` for disks, `/var/lib/libvirt/qemu/nvram/` for NVRAM and `/var/lib/libvirt/swtpm/<UUID>/` for TPM state. These are defaults, not guaranteed actual paths. Use the [storage and metadata reference](VMStorageAndMetadata.md) to discover exact paths and record both name and UUID before deletion.

## Should deleting a VM also delete its network?

Usually not. The example `--network network=default,model=virtio` connects a virtual NIC to an existing network; it does not create a dedicated network for that VM. Libvirt-managed runtime interfaces are cleaned up as the guest stops, while the shared network remains independently managed.

Keep `default`, shared bridges, storage pools, firmware templates, installation ISOs and shared backing images. Only consider deleting a dedicated resource after checking all consumers, including powered-off VMs. A Netplan-managed host bridge is not a disposable VM attachment.

## Does a missing VM entry prove complete deletion?

No. Successful enumeration on the correct host and connection showing neither the old name nor UUID establishes absence from that libvirt domain inventory, not absence of all files or copies. A VM absent under `qemu:///session` might still exist under `qemu:///system` or on the remote server.

Verify each inventoried disk and state object on reachable original storage, then review external snapshots, backups, exports and replicas separately. Searching names alone misses UUID-based state and differently named files. A missing mount, permission error or failed connection is UNKNOWN, not proof of deletion. Logs can remain intentionally without indicating an existing VM.

Use the [verification checklist](VMLifecycleManagement.md#6-verify-deletion) and report the scope honestly: for example, registration and inventoried local resources deleted, backup retained, sanitization not established.

## Is deletion the same as secure erasure?

No. Removing registration and files is ordinary deletion. Secure sanitization addresses whether the underlying data can be recovered. SSD wear levelling, remapped blocks, copy-on-write storage, snapshots and backups can preserve copies despite overwriting a file. `shred` or `--wipe-storage` is not a universal guarantee.

Cryptographic erasure requires a suitable encryption design and accounting for all decryption/recovery material and copies. Deleting one libvirt secret does not prove all keys are gone. Whole-device sanitization affects every resource on that device and is not an ordinary per-VM cleanup step. See [secure erasure limitations](VMLifecycleManagement.md#7-secure-erasure-is-a-separate-requirement).

## Sources

- [Connection URIs - libvirt](https://libvirt.org/uri.html)
- [QEMU/KVM/HVF hypervisor driver - libvirt](https://libvirt.org/drvqemu.html)
- [QEMU documentation: Introduction](https://www.qemu.org/docs/master/system/introduction.html)
- [QEMU documentation: Glossary](https://www.qemu.org/docs/master/glossary.html)
- [Libvirt FAQ](https://wiki.libvirt.org/FAQ.html)
- [virsh command documentation](https://libvirt.org/manpages/virsh.html)
- [Domain XML - libvirt](https://libvirt.org/formatdomain.html)
- [QEMU image tools](https://www.qemu.org/docs/master/tools/qemu-img.html)
- [GNU shred limitations](https://www.gnu.org/software/coreutils/manual/html_node/shred-invocation.html)
