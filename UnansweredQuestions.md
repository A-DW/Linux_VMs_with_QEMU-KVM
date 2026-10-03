# Unanswered Questions

In this document I go through some questions and doubts that I had.

Here are the questions:
 - Why QEMU and KVM, and not one fo the two?
 - Aren't they used for different purposes?
 - What is "``qemu:///system``"?
 - Could "``qemu:///system``" be different with different words?

**QEMU and KVM serve different purposes**, but on Linux they are normally used **together**. QEMU provides the virtual computer; KVM lets QEMU execute the guest's CPU instructions efficiently on the physical CPU.

## Why both?

Think of a VM as needing two major components:

| Component | Responsibility |
| --- | --- |
| QEMU | Contrusct the virtual computer: motherboard, RAM, disks, NICs, USB controllers, firmware, display adpaters and other devices |
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

KVM, conversly, does not provide the complete user-facing virtual computer by itself. It exposes a Linux kernel API, normally through ``/dev/kvm``, which a userspace program such as QEMU uses to execute the VM's virtual CPUs.

Therefore:
```text
QEMU without KVM = complete emulation, but usually slower
KVM without QEMU = acceleration API, but no practical complete VM environment
QEMU with KVM = complete VM with hardware-acccelerated execution
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
```test
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

There are **three slashes** because no remove hostname appears between **//** and **/system**:
```text
qemu://hostname/system     Remote hostname present
qemu:///system             Hostname omitted; local machine
```

The work ``qemu`` selects libvirt's QEMU driver. That driver can manage both software-emulated QEMU machines and KVM-accelerated QEMU machines, which is why the URI is named **qemu**. not necessarily **kvm**.

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

The management service is privileged, although QEMU guest processes are normally subjected to additional user, permissino and security confinement configured by the distribution. Access to the system libvirt API should be trated as highly privileged.

### ``qemu:///session``

```bash
virsh -c qemu:///session list --all
```

This connects to a libvirt instance associated with your user account. Its VMs run with your user's permissions and commonly store their disk images under your home directory. It has fewer permission complications but more restrictions around networking, host devices and boot-tome operation.

Most of your home-lab work should use:
```text
qemu:///system
```

## Are they seperate environments?

Yes. This is very important:
```bash
virsh -c qemu:///system list --all
virsh -c qemu:///session list --all
```

These commands may show **completely different VM lists**.

A VM created under ``qemu:///system`` does nto automatically appear under ``qemu:///session``, even though both connections are on the same computer. They represent separate libvirt management scopes with separate VM definitions, storage configuration and networking capabilities.

This sometimes causes the confusing situation where virt-manager shows a VM but ``virsh  list --all`` does not. Usually, virt-manager and ``virsh`` are connected to different URIs.

Always make the connection explicit while learning:
```bash
virsh -connect qemu:///system list --all
```
The shorter equivalent is:
```bash
virsh -c qemu:///system list --all
```

## Can the URI have different words?

Yes. Different words change the driver, privilege scope, transport or destination.

**Local session instance**
```text
qemu:///session
```

Same QEMU drivers, but your per-user libvirt environment.

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

Virt-manager can use this too, allowing your Ubuntu desktop to manage VMs running on a separate Linux server. libvirt officially supports local, Unix-socker, SSH, TCP and remote variantes of the QEMU connection URI.

### Explicit local Unix transport
```text
qemu+unix:///systen
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

Those do not mean "alternative names for the QEMU/KVM connetion"; they identify different virtualization backends. Libvirt uses connection URIs precisely because it can manage multiple virtualization providers.

## WHy not ``kvm:///system``?

Historically, libvirt has recognized KVM-oriented URI handling, but the modern driver and documented protocol are ``qemu`` because QEMU is the userspace VM monitor being managed. Whether a particular guest uses KVM acceleration is stored in that VM's definition rather than primarily determined by the URI.

You can inspect a VM's definition:
```bash
virsh -c qemu:///system dumpxml VM_NAME | head
```

A KVM-accelerated guest will typically being with something resembling:
```xml
<domain type='kvm'>
```
A software-emulated QEMU guest would instead use:
```xml
<domaing type='qemu'>
```
Therefore, these two concepts answer different questions:
```text
qemu:///system
```
means:
> Which libvirt driver and management environment am I connecting to?
Where as:
```xml
<domain type='kvm?'>
```
means:
> Which execution backend should this particular VM use?

So the precise recommendation is:
> Use **QEMU as the virtual-machine monitor. KVM as QEMU's hardware accelerator, libvirt as the management layer, and connect virt-manager/virsh to ``qemu:///system``**.


















