# Server environment installation

## Introduction

In  the ``README.md`` file is shown the idea of how QEMU/KVM works. Here is a detailed setp-by-step on how to setup QEMU/KVM in a server environment.

## The new architecture

Previously ``virt-manager`` was needed IN the desired Desktop machine where QEMU/KVM were to be. Now, while not necessarily contradicting myself in ``README.md, we don't have to remove the need of ``virt-manager``, we can just change the desired installation destination.

This changes **where the graphical interface should run**, but not the virtualization stack. On Ubuntu Server, is installed **QEMU/KVM + system-wide libvirt**, keep the server headless, and manage it either remotely with **virt-manager over SSH** or through **Cockpit Machines**.

The new architecture:
```bash
Your workstation
├── virt-manager
└── SSH client
        │
        │ qemu+ssh://user@server/system
        ▼
Ubuntu Server
├── libvirt
├── QEMU
├── KVM
├── VM storage
└── Virtual networks
```

I **do not** need:
- Ubuntu Desktop on the server
- ``virt-manager`` installed on the server
- A monitor permanently attached
- Proxmox
- To write raw QEMU commands manually

Ubuntu even explicitly recommends installing virt-manager on a workstation rather than on a production server, and it supports managing a remote libvirt host through SSH.

## Before isntalling

This assumes Ubuntu Server is installed **direcly on physical hardware**.

Check whether CPU virtualization is available:
```bash
sudo apt update
sudo apt install cpu-checker
sudo kvm-ok
```
Expected result:
```text
INFO: /dev/kvm exists
KVM acceleration can be used
```

Ubuntu recommends ``kvm-ok`` as the initial KVM hardware-support check.

You can also inspect the CPU flags:
```bash
lscpu | grep -E 'Virtualization|Hypervisor'
```
Typical output:
```text
Virtualization: AMD-V
```
or
```text
Virtualization: VT-x
```

If the Ubuntu server is itself a VM, the outer hypervisor must provide nested virtualization. Otherwise, ``/dev/kvm`` may be unavailable and you inner VMs would either fail or use very slow software emulation.

## Install the server stack

On the Ubuntu Server:
```bash
sudo apt update

sudo apt install qemu-kvm qemu-utils libvirt-daemon-system libvirt-clients virtinst ovmf swtpm swtpm-tools
```

The critical packages are:
| Package |  Purpose |
| --- | --- |
| ``qemu-kvm`` | QEMU system virtualization with KVM support |
| ``libvirt-daemon-system`` | System-wide VM management |
| ``libvirt-clients`` | Provides ``virsh`` and related clients |
| ``virtinst`` | Provides ``virt-install`` for creating VMs |
| ``qemu-utils`` | Disk tools such as ``qemu-img`` |
| ``ovmf`` | UEFI firmware for modern guests |
| ``swtpm`` | Virtual TPM, especially useful for Windows 11 |

Ubuntu documents ``qemu-kvm`` and ``libvirt-daemon-system`` as the core Ubuntu Server installation.

Add your administrative account to the relevant groups:
```bash
sudo adduser "$USER" libvirt
sudo adduser "$USER" kvm
```
Then **log out of SSH and reconnect** so the new group memberships take effect.

Be selective about membership in ``libvirt``: users capable of defining arbitrary system VMs may given access to sensitive host resources. Treat it as an administrative group.

## Verify the server

After reconnecting:
```bash
id
```
You should see ``libvirt`` and ``kvm`` in the group list.

Then run:
```bash
virt-host-valuable qemu
```

Check the system-wide libvirt connection:
```bash
virsh -c qemu:///system list --all
```

Expected initially:
```text
 Id   Name   State
--------------------
```
An empty list is not an error — it means libvirt is working, but no VMs exist yet.

Check ``/dev/kvm``:
```bash
ls -l /dev/kvm
```

Check libvirt networking:
```bash
virsh -c qemu:///system net-list --all
```

You will normally want a network resembling:
```text
 Name      State    Autostart   Persistent
------------------------------------------------
 default   active   yes         yes
```

If ``default`` exists but is inactive:
```bash
virsh -c qemu:///system net-start default
virsh -c qemu:///system net-autostart default
```





























