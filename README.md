# Linux_VMs_with_QEMU-KVM

This is my learning path experience to using Virtual Machines (VMs) on Linux with QEMU/KVM. This is not an official document.

## Introduction

If I'd want to run VMs in Windows, I could run something as simple as VirtualBox and it would be almost out-of-the-box to use. However in Linux, it's not that simple as executing a .exe file to install a software. It requires some extra steps.

There are some virtualization options in Linux: KVM, QEMU, libvirt, virt-manager, virsh, virt-install, GNOME Boxes, VirtualBox and Proxmox VE. But what are these? What are they for? And most important, what do I actually need to run VMs on Linux?

For those questions to be answered, we need to first understand what they are:

| Component | Role | Should it be used directly? |
| --- | --- | --- |
| KVM | Linux kernel facility that exposes the CPU's hardware virtualization capabilities to userspace. | Usually no: applictions such as QEMU use it. |
| QEMU | Runs the VM and supplies its virtual machine model, including CPU, memory, disk, network and other devices. It can emulate a CPU in software or use KVM to run compatible guest code on the real CPU. | Normally not for everyday desktop VM management. Direct QEMU commands are useful for specialist for experimental  setups. |
| libvirt | Management API and service for VMs, virtual networks and storage. ``virsh``, ``virt-install`` and ``virt-manager`` all use it. | Yes, indirectly through a frontend; learn its concepts over time. |
| virt-manager | Detailed graphical frontend for libvirt, capable of creating and managing local or remote VMs. | Yes — this should be the primary desktop application. |
| virsh | Command-line manager supplied by libvirt; it can list, start, stop, edit and automate guests. | Yes, after learnin the GUI basics. |
| virt-install | Command-line VM-creation tool included with the virt-manager tools. | Yes, when repeatability and scripting matter. |
| GNOME Boces | Simplified GUI that also uses QEMU/KVM, libivrt and SPICE. | Optional. Convenient for disposable test VMs, but less suitable for learning advanced VM adminsitration. |
| VirtualBox | Self-contained cross-platform desktop virtualization product. | Keep oly if an existing workflow, appliance or course specifically requires it. |
| Proxmox VE | A complete, dedicated virtualization platform, a Linux destribution, rather than a normal (client) Ubuntu desktop application. | Not for the stated requirement. It becomes relevant later for a separate always-on lab host. |

I could just use VirtualBox, as I were before on Windows, but given that KVM is part of the Linux kernel, I can use the closest capability to bare-metal CPU's hardware virtualization possible. Not to mention that while on Windows it may do the work for something like a side project, the feedback is not that good when it comes to Linux.

So why not try something new and works better?

For that reason, lets use **QEMU/KVM with libvirt and virt-manager**. In practical terms, **virt-manager** is the application you open; QEMU/KVM is the virtualization engine underneath it.

---

## How the stack works

```test
virt-manager        → Graphical interface
virsh               → Command-line interface
       ↓
libvirt             → Manages VMs, networks and storage
       ↓
QEMU                → Provides the virtual machine and virtual hardware
       ↓
KVM                 → Uses CPU hardware virtualization for acceleration
       ↓
Intel VT-x / AMD-V  → Uses CPU hardware virtualization for acceleration
```

- **KVM** is part of the Linux kernel, not a standalone GUI/program.
- **QEMU** creates and runs the virtual hardware.
- **libvirt** manages QEMU/KVM machines consistently.
- **virt-manager** gives you a VirtualBox-like interface with substantially more control.

QEMU can emulate complete machines by itself, but when paired with KVM it can execute compatible guest code directly through the host CPU's virtualization capabilities. Virt-manager is Ubuntu's documented graphical interface for managing local and remote libvirt vitual machines.

## Desktop environment installation

### Recommended installation

On a current Ubuntu workstation(/client), install the official packages:

```bash
sudo apt update
sudo apt install qemu-kvm libvirt-daemon-system libvirt-clients virt-manager virtinst ovmf swtpm swtpm-tools
```

Ubunt's current documentaion lists ``qemu-kvm`` and ``libvirt-daemon-system`` as the core installation and ``virt-manager`` as its graphical manager. ``ovmf`` supplies UEFI firmware for x86-64 guests, while ``swtpm`` supplies an emulated TPM when a modern guest such as Windows 11 needs one.

Give your account VM-management permissions to the management groups:
```bash
sudo usermod -aG libvirt,kvm "$USER"
```
Then **log out completely and log back in**.

Check the host afterward:
```bash
virt-host-validate qemu
virsh --connect qemu:///system list --all
```

``virt-host-validate`` verifies whether the machine is suitably configured for a selected libvirt hypervisor driver. If KVM validation fails, confirm that Intel VT-x or AMD-V/SVM is enabled in the UEFI/BIOS and that ``/dev/kvm`` exists.

Launch the GUI:
```bash
virt-manager
```

Use the **QEMU/KVM system connection** — normally displayed as **QEMU/KVM** or **localhost (QEMU)** — rathar then building a separate user-session setup initially. The system connection is the conventional choice for server-like guests, host-boot autostart and managed virtual networking.
```text
QEMU/KVM
localhost (QEMU)
qemu:///system
```





















