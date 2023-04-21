## **Table Of Contents**
* **[IOMMU Setup](#enable--verify-iommu)**
* **[Installing Packages](#install-required-tools)**
* **[Enabling Services](#enable-required-services)**
* **[Guest Setup](#setup-guest-os)**
* **[Attching PCI Devices](#attaching-pci-devices)**
* **[Keyboard/Mouse Passthrough](#keyboardmouse-passthrough)**
* **[Video Card Virtualisation Detection](#video-card-driver-virtualisation-detection)**
* **[Audio Passthrough](#audio-passthrough)**

### **Enable & Verify IOMMU**
***BIOS Settings*** \
Enable ***Intel VT-d*** or ***AMD-Vi*** and disabled ***CSM*** support in BIOS settings. If these options are not present, it is likely that your hardware does not support IOMMU.

Disable ***Resizable BAR Support*** in BIOS settings. 
Cards that support Resizable BAR can cause problems with black screens following driver load if Resizable BAR is enabled in UEFI/BIOS. There doesn't seem to be a large performance penalty for disabling it, so turn it off for now until ReBAR support is available for KVM. 

***Set the kernel paramater depending on your CPU.*** \
For GRUB user, edit grub configuration.
| /etc/default/grub |
| ----- |
| `GRUB_CMDLINE_LINUX_DEFAULT="... intel_iommu=on iommu=pt ..."` |
| OR |
| `GRUB_CMDLINE_LINUX_DEFAULT="... amd_iommu=on iommu=pt ..."` |

***Generate grub.cfg***
```sh
grub-mkconfig -o /boot/grub/grub.cfg
```
Reboot your system for the changes to take effect.

***To verify IOMMU, run the following command.***
```sh
dmesg | grep IOMMU
```
The output should include the message `Intel-IOMMU: enabled` for Intel CPUs or `AMD-Vi: AMD IOMMUv2 loaded and initialized` for AMD CPUs.

To view the IOMMU groups and attached devices, run the following script:
```sh
#!/bin/bash
shopt -s nullglob
for g in `find /sys/kernel/iommu_groups/* -maxdepth 0 -type d | sort -V`; do
    echo "IOMMU Group ${g##*/}:"
    for d in $g/devices/*; do
        echo -e "\t$(lspci -nns ${d##*/})"
    done;
done;
```

When using passthrough, it is necessary to pass every device in the group that includes your GPU. \
You can avoid having to pass everything by using [ACS override patch](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF#Bypassing_the_IOMMU_groups_(ACS_override_patch)).

### **Install required tools**
<details>
  <summary><b>Gentoo Linux</b></summary>

  ```sh
  emerge -av qemu virt-manager libvirt ebtables dnsmasq
  ```
</details>

<details>
  <summary><b>Arch Linux</b></summary>

  ```sh
  pacman -S qemu libvirt edk2-ovmf virt-manager dnsmasq ebtables
  ```
</details>

<details>
  <summary><b>Fedora</b></summary>

  ```sh
  dnf install @virtualization
  ```
</details>

<details>
  <summary><b>Ubuntu</b></summary>

  ```sh
  apt install qemu-kvm qemu-utils libvirt-daemon-system libvirt-clients bridge-utils virt-manager ovmf
  ```
</details>

### **Enable required services**
<details>
  <summary><b>SystemD</b></summary>

  ```sh
  systemctl enable --now libvirtd
  ```
</details>

<details>
  <summary><b>OpenRC</b></summary>

  ```sh
  rc-update add libvirtd default
  rc-service libvirtd start
  ```
</details>

Sometimes, you might need to start default network manually.
```sh
virsh net-start default
virsh net-autostart default
```

### **Setup Guest OS**
***NOTE: You should replace win10 with your VM's name where applicable*** \
You should add your user to ***libvirt*** group to be able to run VM without root. And, ***input*** and ***kvm*** group for passing input devices.
```sh
usermod -aG kvm,input,libvirt username
```

Download [virtio](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso) driver. \
Launch ***virt-manager*** and create a new virtual machine. Select ***Customize before install*** on Final Step. \
In ***Overview*** section, set ***Chipset*** to ***Q35***, and ***Firmware*** to ***UEFI*** \
In ***CPUs*** section, set ***CPU model*** to ***host-passthrough***, and ***CPU Topology*** to whatever fits your system. \
For ***SATA*** disk of VM, set ***Disk Bus*** to ***virtio***. \
In ***NIC*** section, set ***Device Model*** to ***virtio*** \
Add Hardware > CDROM: virtio-win.iso \
Now, ***Begin Installation***. Windows can't detect the ***virtio disk***, so you need to ***Load Driver*** and select ***virtio-iso/amd64/win10*** when prompted. \
After successful installation of Windows, install virtio drivers from virtio CDROM. You can then remove virtio iso.

### **Attaching PCI devices**
Remove Channel Spice, Display Spice, Video QXL, Sound ich* and other unnecessary devices. \
Now, click on ***Add Hardware***, select ***PCI Devices*** and add the PCI Host devices for your GPU's VGA and HDMI Audio.

### **Keyboard/Mouse Passthrough**
In order to be able to use keyboard/mouse in the VM as hardware via PCI devices. Go to vm settgins then click on add hardware and then check PCI and click on USB Host controller (or something similar). 

### **Audio Passthrough**
To passthrough add it's hardware, so open virt-manager settings then go to add hardware then go to PCI devices and choose the HD audio Controller (or something similar) and add it and all of it's IOMMU group (you can check the group via):
```sh
#!/bin/bash
shopt -s nullglob
for g in `find /sys/kernel/iommu_groups/* -maxdepth 0 -type d | sort -V`; do
    echo "IOMMU Group ${g##*/}:"
    for d in $g/devices/*; do
        echo -e "\t$(lspci -nns ${d##*/})"
    done;
done;
```


### **Video card driver virtualisation detection**
Video Card drivers refuse to run in Virtual Machine, so you need to spoof Hyper-V Vendor ID.
<table>
<tr>
<th>
virsh edit win10
</th>
</tr>

<tr>
<td>

```xml
...
<features>
  ...
  <hyperv>
    ...
    <vendor_id state='on' value='whatever'/>
    ...
  </hyperv>
  ...
</features>
...
```

</td>
</tr>
</table>

NVIDIA guest drivers also require hiding the KVM CPU leaf:
<table>
<tr>
<th>
virsh edit win10
</th>
</tr>

<tr>
<td>

```xml
...
<features>
  ...
  <kvm>
    <hidden state='on'/>
  </kvm>
  ...
</features>
...
```

</td>
</tr>
</table>


### **See Also**
> [Single GPU Passthrough Troubleshooting](https://docs.google.com/document/d/17Wh9_5HPqAx8HHk-p2bGlR0E-65TplkG18jvM98I7V8)<br/>
> [Single GPU Passthrough by joeknock90](https://github.com/joeknock90/Single-GPU-Passthrough)<br/>
> [Single GPU Passthrough by YuriAlek](https://gitlab.com/YuriAlek/vfio)<br/>
> [Single GPU Passthrough by wabulu](https://github.com/wabulu/Single-GPU-passthrough-amd-nvidia)<br/>
> [ArchLinux PCI Passthrough](https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF)<br/>
> [Gentoo GPU Passthrough](https://wiki.gentoo.org/wiki/GPU_passthrough_with_libvirt_qemu_kvm)<br/>
> [VFIO Discord](https://discord.gg/FtTJ5g4mxS)<br/>
> [VFIO subreddit](https://reddit.com/r/VFIO)<br/>
