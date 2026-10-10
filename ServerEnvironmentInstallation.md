# Server environment installation

## Introduction

In the [README.md](README.md) is shown the idea of how QEMU/KVM works. Here is a detailed step-by-step on how to set up QEMU/KVM in a server environment.

## The new architecture

Previously ``virt-manager`` was needed IN the desired Desktop machine where QEMU/KVM were to be. Now, while not necessarily contradicting myself in [README.md](README.md), we don't have to remove the need of ``virt-manager``, we can just change the desired installation destination.

This changes **where the graphical interface should run**, but not the virtualization stack. On Ubuntu Server, install **QEMU/KVM + system-wide libvirt**, keep the server headless, and manage it either remotely with **virt-manager over SSH** or through **Cockpit Machines**.

The new architecture:
```text
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

This assumes Ubuntu Server is installed **directly on physical hardware**.

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

If the Ubuntu server is itself a VM, the outer hypervisor must provide nested virtualization. Otherwise, ``/dev/kvm`` may be unavailable and your inner VMs would either fail or use very slow software emulation.

## Install the server stack

On the Ubuntu Server:
```bash
sudo apt update

sudo apt install qemu-kvm qemu-utils libvirt-daemon-system libvirt-clients virtinst ovmf swtpm swtpm-tools
```

The critical packages are:

| Package | Purpose |
| --- | --- |
| ``qemu-kvm`` | QEMU system virtualization with KVM support |
| ``libvirt-daemon-system`` | System-wide VM management |
| ``libvirt-clients`` | Provides ``virsh`` and related clients |
| ``virtinst`` | Provides ``virt-install`` for creating VMs |
| ``qemu-utils`` | Disk tools such as ``qemu-img`` |
| ``ovmf`` | UEFI firmware for modern guests |
| ``swtpm`` | Virtual TPM, especially useful for Windows 11 |

Ubuntu documents ``qemu-kvm`` and ``libvirt-daemon-system`` as the core Ubuntu Server installation.

Add your administrative account to the ``libvirt`` group:
```bash
sudo adduser "$USER" libvirt
```
Then **log out of SSH and reconnect** so the new group membership takes effect.

Treat the ``libvirt`` group as administrative, because membership is effectively near-root on the host: a member can define arbitrary system VMs, which may give access to sensitive host resources such as disks and devices. Only add accounts you fully trust.

The ``kvm`` group is not required for VMs managed through ``qemu:///system``, because libvirt runs those guests under its own service account. It is only relevant when your own user starts QEMU directly, so following the principle of least privilege it is not added here.

## Verify the server

After reconnecting:
```bash
id
```
You should see ``libvirt`` in the group list.

Then run:
```bash
virt-host-validate qemu
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
| **virt-manager over SSH** | Your Linux workstation | Best overall graphical interface |
| **Cockpit Machines** | Web service on the Ubuntu Server | Convenient browser-based management |
| **virsh + virt-install** | SSH terminal | Learning, automation, and troubleshooting |

My recommendation is:
1) Start with **virt-manager remotely** if your client computer runs Linux.
2) Add **Cockpit Machines** if you want browser-based access.
3) Gradually learn ``virsh`` and ``virt-install``.

## Remote virt-manager

On an Ubuntu desktop or another Debian-based Linux workstation:
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

Once key-based login works, consider disabling password logins on the server by setting ``PasswordAuthentication no`` in ``/etc/ssh/sshd_config`` (or a drop-in file under ``/etc/ssh/sshd_config.d/``), then reloading SSH. Keep your current session open and test a new login from a second terminal first, so a mistake does not lock you out.

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

## Cockpit alternative

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
Do not expose port ``9090`` directly to the public internet. Restrict it to your LAN, management VLAN, WireGuard/Tailscale network, or SSH tunnel.

A documented warning is not enforcement, so also restrict the port with a firewall rule. For example, with UFW, allow only your management subnet (replace ``MANAGEMENT_SUBNET`` with your own, such as ``192.168.1.0/24``):
```bash
sudo ufw allow from MANAGEMENT_SUBNET to any port 9090 proto tcp
sudo ufw status numbered
```
Make sure SSH access is allowed before enabling UFW on a remote server, otherwise you can lock yourself out.

For an SSH tunnel:
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

First list the OS identifiers known to your system, and pick the one matching your guest:
```bash
virt-install --osinfo list | grep -i ubuntu
```

Then create the VM (replace ``ubuntu24.04`` with the identifier matching your ISO):
```bash
sudo virt-install \
  --connect qemu:///system \
  --name lab-ubuntu-01 \
  --memory 4096 \
  --vcpus 2 \
  --cpu host \
  --disk path=/var/lib/libvirt/images/lab-ubuntu-01.qcow2,size=30,format=qcow2,bus=virtio \
  --cdrom /var/lib/libvirt/boot/ubuntu-server.iso \
  --osinfo ubuntu24.04 \
  --network network=default,model=virtio \
  --graphics spice \
  --boot uefi \
  --noautoconsole
```

A specific ``--osinfo`` value lets virt-install choose appropriate virtual hardware defaults for the guest, which a ``generic`` value does not.

Then inspect it:
```bash
virsh -c qemu:///system list --all
```

Start or stop it:
```bash
virsh -c qemu:///system start lab-ubuntu-01
virsh -c qemu:///system shutdown lab-ubuntu-01
```

Enable automatic startup with the physical server:
```bash
virsh -c qemu:///system autostart lab-ubuntu-01
```

Ubuntu documents ``virsh`` for VM lifecycle management, including starting guests and enabling autostart.

Access the graphical installer through remote virt-manager or Cockpit. Ubuntu also supports building guests from QCOW cloud images with ``virt-install`` and cloud-init, which will be useful once you progress from manual installations to repeatable deployments.

## Storage warning

By default libvirt stores VM disk images under ``/var/lib/libvirt/images``, which normally lives on the root filesystem. Growing QCOW2 images can fill it and affect the whole server. For anything beyond a small lab, create a dedicated storage pool on a separate disk or LVM volume.

## Networking choice

For your first VMs, select:
```text
Virtual network: default
Mode: NAT
```

Conceptually:
```text
Physical LAN
    │
Ubuntu Server
    │
libvirt NAT network
    │
VMs: 192.168.122.0/24
```
The VMs can ordinarily access the LAN and Internet, but other LAN devices cannot initiate connections to them without forwarding or routing configuration.

Later, if you want each VM to have its own IP address from your physical LAN, create a **Linux bridge**, such as ``br0``:
```text
Physical NIC ── br0 ── Ubuntu Server
                    ├── VM 1
                    ├── VM 2
                    └── VM 3
```
Do not begin by changing the server's main network interface remotely unless you have console access, because a malformed Netplan bridge configuration can disconnect the server.

## Troubleshooting

| Symptom | Likely cause | What to check |
| --- | --- | --- |
| ``/dev/kvm`` does not exist | Virtualization disabled in UEFI/BIOS, or the server is a VM without nested virtualization | ``sudo kvm-ok``, ``lscpu \| grep Virtualization``, UEFI/BIOS settings, nested virtualization on the outer hypervisor |
| ``virsh`` says permission denied | Your account is not in the ``libvirt`` group, or you have not logged in again since being added | ``id``, then log out and reconnect |
| ``qemu+ssh`` connection fails | SSH key or user problem, or libvirt not running on the server | ``ssh youruser@server-ip``, then ``systemctl status libvirtd`` on the server |
| VM has no network | The ``default`` libvirt network is inactive | ``virsh -c qemu:///system net-list --all``, then ``net-start default`` and ``net-autostart default`` |
| virt-manager shows a VM but ``virsh list --all`` does not | The two tools are connected to different URIs | Always pass ``-c qemu:///system`` (or the ``qemu+ssh`` URI) explicitly |

## Sources

- [Libvirt - Ubuntu Server documentation](https://ubuntu.com/server/docs/how-to/virtualisation/libvirt/)
- [Virtual Machine Manager - Ubuntu Server documentation](https://ubuntu.com/server/docs/how-to/virtualisation/virtual-machine-manager/)
- [Launch QCOW images using libvirt - Ubuntu](https://ubuntu.com/docs/public-images/public-images-how-to/launch-with-libvirt/)
- [Libvirt SSH setup](https://wiki.libvirt.org/SSHSetup.html)
- [Virtual Machines - Cockpit Project](https://cockpit-project.org/guide/195/feature-virtualmachines.html)
