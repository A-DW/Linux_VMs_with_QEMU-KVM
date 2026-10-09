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

## Before installing

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

## Management options

You have three reasonable management methods.

| Method | Where it runs | Best use |
| --- | --- | --- |
| **virt-manager over SSH** | You Linux workstation | Best overall graphical interface |
| **Cockpit Machines** | Web service on the Ubuntu Server | Convenient browser-based management |
| **virsh + virt-install** | SSH terminal | Learning, automation, and troubleshooting |

My recommendation is:
1) Start with **virt-manager remotely** if you client computer runs Linux.
2) Add **Cockpit Machines** if you want browser-based access.
3) Gradually learn ``virsh`` and ``virt-install``.

## Remote virt-maanger

On an UBuntu desktop or another Debian-based Linux workstation:
```bash
sudo apt install virt-manager
```
Do **not** install it on the headless server.

Configure SSH keys:
```bash
ssh-keygen -t ed25519
ssh-copy-id youruser@server-ip
```

Test normal SSH access:
```bash
ssh youruser@server-ip
```

Then test libvirt remotely:
```bash
virsh -c qemu+ssh://youruser@server-ip/system list --all
```

Start virt-manager using that same connection:
```bash
virt-manager -c qemu+ssh://youruser@server-ip/system
```

Ubuntu documents this exact management model: virt-manager runs on a graphical workstation and connects to the server's system libvirt instance through SSH keys.

Notice the difference:
```text
qemu:///system
```
means:
> Connect to system libvirt on this local machine.

Whereas:
```text
qemu+ssh://youruser@server-ip/system
```
means:
> Connect over SSH to the system libvirt instance on another machine.

The VMs are still running on the Ubuntu Server. Virt-manager merely displays and controls them remotely.

## Cockpit altenative

If your management computer is Windows or you want something closer to Proxmox's browser-based experience — without replacing Ubuntu Server — install Cockpit:

```bash
sudo apt install cockpit cockpit-machines
sudo systemctl enable --now cockpit.socket
```

Cockpit Machines manages the same QEMU/libvirt VMs, rather than creating a separate virtualization environment.

Open:
```text
https://SERVER-IP:9090
```
Do not expose port ``9090`` directly to the public internet. Restrict it to your LAN, managament VLAN, WireGuard/Tailscale network, or SSH tunnel.

For an SSH tunel:
```bash
ssh -L 9090:localhost:9090 youruser@server-ip
```

Then open locally:
```text
https://localhost:9090
```
Cockpit is easier initially, but virt-manager normally exposes more low-level VM configuration. Both can manage VMs under the same ``qemu:///system`` environment.

## Create your first VM

For learning, create an Ubuntu Server VM inside the Ubuntu Server host.

Organize your ISO images:
```bash
sudo mkdir -p /var/lib/libvirt/boot
sudo cp ~/ubuntu-server.iso /var/lib/libvirt/boot
```

Then create the VM:
```bash
sudo virt-install \
  --connect qemu:///system \
  --name lab-ubuntu-01 \
  --memory 4096 \
  --vcpus 2 \
  --cpu host \
  --disk path=/var/lib/libvirt/images/lab-ubuntu-01.qcow2,size=30,foanat=qcow2,bus=virtio \
  --cdrom /var/lib/libvirt/boot/ubuntu-server.iso \
  --osinfo detect=on,name=generic \
  --network network=default,model=virtio \
  --graphics spice \
  --boot uefi \
  --noautoconsole
```

Then inspect it:
```bash
virsh -c qemu:///system list --all
```

Start or stop it:
```bash
virsh -c qemu:///system start lab-ubuntu-01
virsh -c qemu:///system shutdown lab-ubuntu-01
```

Enable automatic startup with the physical server
```bash
virsh -c qemu:///system autostart lab-ubuntu-01
```

Ubuntu documents ``virsh`` for VM lifecycle management, including starting guests and enabling autostart.

Access the graphical installer through remote virt-manager or Cockpit. Ubuntu also supports building guests from QCOW cloud images with ``virt-install`` and cloud-init, which will be useful once you progress from manual installations to repeatable deployments.





























