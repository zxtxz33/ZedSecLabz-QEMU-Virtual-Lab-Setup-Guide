# ZedSecLabz: QEMU/KVM Virtual Lab Deployment Manual

## Introduction

This manual outlines the process of establishing a local virtualization environment using QEMU/KVM and libvirt, configured for headless operation via SPICE. It’s tailored for penetration testers and lab operators who need high-performance VM management without a graphical interface.

---

## System Requirements

* OS: Arch Linux or any systemd-compatible distro
* Virtualization support enabled in BIOS/UEFI (Intel VT-x / AMD-V)
* Required packages:

```bash
sudo pacman -S qemu virt-manager virt-viewer dnsmasq vde2 bridge-utils openbsd-netcat libvirt spice-gtk ebtables iptables
```

Initialize the libvirt daemon:

```bash
sudo systemctl enable --now libvirtd
```

**Reference Documentation**:

* [https://wiki.archlinux.org/title/KVM](https://wiki.archlinux.org/title/KVM)
* [https://wiki.archlinux.org/title/Libvirt](https://wiki.archlinux.org/title/Libvirt)
* [https://www.qemu.org/](https://www.qemu.org/)
* [https://virt-manager.org/](https://virt-manager.org/)
* [https://www.spice-space.org/](https://www.spice-space.org/)

---

## Storage Pool Configuration

### Create Storage Directory

```bash
sudo mkdir -p /var/lib/libvirt/images/zedsec
sudo chown -R libvirt-qemu:kvm /var/lib/libvirt/images/zedsec
```

### Register the Storage Pool

```bash
virsh pool-define-as zedsec_pool dir --target /var/lib/libvirt/images/zedsec
virsh pool-build zedsec_pool
virsh pool-start zedsec_pool
virsh pool-autostart zedsec_pool
```

Verify:

```bash
virsh pool-list --all
```

---

## Default NAT Network Setup

### Enable Required Services

```bash
sudo systemctl start virtnetworkd.socket
sudo systemctl enable virtnetworkd.socket
```

For Arch Linux systems, also ensure:

```bash
sudo systemctl enable --now virtqemud.socket virtproxyd.socket virtstoraged.socket
```

### Manual Network Definition (if missing)

```bash
cat <<EOF | sudo tee /tmp/default-net.xml > /dev/null
<network>
  <name>default</name>
  <forward mode='nat'/>
  <bridge name='virbr0' stp='on' delay='0'/>
  <ip address='192.168.122.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.122.2' end='192.168.122.254'/>
    </dhcp>
  </ip>
</network>
EOF

sudo virsh net-define /tmp/default-net.xml
sudo virsh net-start default
sudo virsh net-autostart default
```

Check:

```bash
virsh net-list --all
```

---

## Headless Virtual Machine Startup with SPICE

### Start the VM

```bash
sudo virsh start <vm-name>
```

### Confirm SPICE Configuration

```bash
sudo virsh dumpxml <vm-name> | grep spice
```

Expected output:

```xml
<graphics type='spice' port='5900' autoport='no' listen='0.0.0.0'>
```

Modify if necessary:

```bash
sudo virsh edit <vm-name>
```

---

## Connecting to a Running VM via SPICE

### From Localhost

```bash
remote-viewer spice://127.0.0.1:5900
```

### Remote Access Over SSH

```bash
ssh -L 5900:localhost:5900 user@remotehost
remote-viewer spice://127.0.0.1:5900
```

---

## Alternate: Raw QEMU Execution with SPICE (No libvirt)

```bash
qemu-system-x86_64 \
  -hda /var/lib/libvirt/images/zedsec/yourdisk.qcow2 \
  -m 2048 \
  -enable-kvm \
  -vga qxl \
  -spice port=5900,disable-ticketing \
  -daemonize \
  -display none
```

Access using:

```bash
remote-viewer spice://127.0.0.1:5900
```

---

## Additional Operational Considerations

* Use host-only or NAT-isolated networking for security
* Always snapshot VMs before testing payloads or malware
* Prefer CLI tools (`virsh`, `remote-viewer`) in remote/headless ops
* Use `virt-manager` only when necessary for GUI-based inspection

---

## Official Sources and Legal Reference

* QEMU: [https://www.qemu.org/](https://www.qemu.org/)
* SPICE Protocol: [https://www.spice-space.org/](https://www.spice-space.org/)
* Libvirt API: [https://libvirt.org/](https://libvirt.org/)
* virt-manager: [https://virt-manager.org/](https://virt-manager.org/)
* remote-viewer: [https://gitlab.gnome.org/GNOME/virt-viewer](https://gitlab.gnome.org/GNOME/virt-viewer)
* Fedora VM Docs: [https://docs.fedoraproject.org/en-US/quick-docs/creating-and-using-a-virtual-machine/](https://docs.fedoraproject.org/en-US/quick-docs/creating-and-using-a-virtual-machine/)
* Arch Wiki on Libvirt: [https://wiki.archlinux.org/title/Libvirt](https://wiki.archlinux.org/title/Libvirt)

> All tools listed are open-source and governed by their respective licenses. This guide is intended for lawful, ethical use within sandboxed or educational environments only.
