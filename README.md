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

I could just use VirtualBox, as I were before on Windows, but given that KVM is part of the Linux kernel, I can use the closest capability to bare-metal CPU's hardware virtualization possible. In my own experience, VirtualBox on Linux felt heavier on resources than QEMU/KVM, so I decided to try the native Linux stack.

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
KVM                 → Linux kernel module that exposes hardware virtualization to QEMU
       ↓
Intel VT-x / AMD-V  → CPU hardware virtualization extensions
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

Give your account VM-management permissions by adding it to the ``libvirt`` group:
```bash
sudo usermod -aG libvirt "$USER"
```
Then **log out completely and log back in**.

Membership in the ``libvirt`` group is effectively administrative: a member can define arbitrary system VMs, which may expose sensitive host resources. Only add trusted accounts. The ``kvm`` group is not needed for VMs managed through ``qemu:///system``, because libvirt runs those guests under its own service account; it is only relevant when your own user starts QEMU directly.

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

### First VM settings

For the initial Linux VM, let's use:
- Connection: QEMU/KVM system
- Installation source: Local ISO
- Firmware: UEFI
- Disk format: QCOW2, dynamically allocated
- Disk bus: VirtIO or VirtIO-SCSI
- Network source: Virtual network ``default`` — NAT
- Network model: VirtIO
- Display: SPICE
- CPU configuration: Default initially; try host-passthrough later
- Networking: Do not configure a bridge yet unless the VM must appear directly on your physical LAN

For a first lab, **NAT is the correct network choice**. The VM receives outbound network access while remaining behind a libvirt-managed virtual network. Use a Linux bridge only when a guest must appear as a separate machine directly on the physical LAN; bridging is an additional netwokring skill, not a necessary first step.

#### Ubuntu guests

Inside an Ubuntu guest, install:
```bash
sudo apt update
sudo apt install qemu-guest-agent spice-vdagent
sudo systemctl enable --now qemu-guest-agent
```

The QEMU guest agent provides a controlled host-to-guest communication channel, while SPICE components improve interactive desktop integration. QEMU's documentation and enterprise virtualization guidance identify the guest agent as the component through which a host can issue supported commands to a guest.

#### Windows guests

Windows works well under this stack, but it benefits strongly from **VirtIO drivers** for virtual storage and networking. The Fedora project maintains the Windows VirtIO drivers because Microsoft does not include them with Windows.

For an uncomplicated first Windows installation:
 1. Create the VM with virt-manager and select the detected Windows version.
 2. Use UEFI and add and emulated TPM 2.0 for Windows 11.
 3. Attach both the Windows installation ISO and the stable ``virtio-win.iso``.
 4. If Windows Setup cannot see a VirtIO disk, choose **Load driver** and select the matching storage driver from the VirtIO ISO.
 5. After Windows boots, install the full VirtIO guest tools and QEMU guest agent.

VirtIO replaces slower fully emulated disk and network devices with paravirtualized devices designed for virtual machines. The official virtio-win project publishes signed binary drivers in ISO form for QEMU/KVM Windows guests.

For troubleshooting, begin with conservative virtual devices rather than changing several advanced options simultaneously. If a Windows guest freezes, record the VM XML and host logs, then test display, VirtIO drivers, firmware and CPU configuration one variable at a time.

## Server environment installation

Because I want to use an already existing Ubuntu server I have, I have to setup QEMU/KVM differently.

For example, installing ``virt-manager``, a graphical frontend on a server environment is not only unnecessary, but also it does not serve its purpose. This changes **where the graphical interface should run**, but not the virtualization stack.

I explain this different approach in detail in [ServerEnvironmentInstallation.md](ServerEnvironmentInstallation.md).

## Practical command sheet

```bash
# Host validation
virt-host-validate qemu

# Show every system VM
virsh -c qemu:///system list --all

# Start and gracefully stop a guest
virsh -c qemu:///system start VM_NAME
virsh -c qemu:///system shutdown VM_NAME

# Enable VM startup when the host boots
virsh -c qemu:///system autostart VM_NAME

# Inspect the VM definition
virsh -c qemu:///system dumpxml VM_NAME

# Show virtual networks
virsh -c qemu:///system net-list --all

# Show storage pools
virsh -c qemu:///system pool-list --all
```

These commands use the same libvirt-managed machines visible in virt-manager, so GUI and CLI learning reinforce each other instead of creating two separate VM environments.

## What not to choose

### QEMU alone

Do not begin by manually writing large ``qemu-system-x86_64`` commands. That appreaoch is valuable for understanding experiments, but it means personally managing arguments for storage, networking, firmware, display and lifecycle. QEMU can also emulate other CPU architectures entirely in software, but for same-architecture guests (for example an x86-64 guest on an x86-64 host) KVM acceleration should be used for ordinary work.

### KVM alone

KVM is not a VirtualBox-like desktop application. It is a kernel virtualization facility used by a userspace virtual-machine monitor such as QEMU: Asking whether to use "KVM or QEMU" is therefore similar to asking whether to use a graphics driver or the application that draws the interface: in the setup, both participate.

### GNOME Boxes as the main tool

Boxes uses much of the same backend stack and is useful for quick-tests, but its interface intentionally omits many advanced options. It is not the strongest choice for learning virtual networks, storage pools, firmware, device models, and remote hypervisors or repeatable deployment.

### Proxmox on the workstation

Proxmox is appropriate when a computer's primary purpose is to act as a dedicated virtualization server with browser-based, centralized administration. It is unnecessary when the objective is to keep Ubuntu as the daily desktop and run local VMs inside it. If a separate home-lab host is added later, Proxmox can be evaluated independently without changing the recommended Ubuntu workstation stack.
