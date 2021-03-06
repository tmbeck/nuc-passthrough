# Intel NUC NUC7i7BNH IGD Passthrough

In this example, we successfully passthrough the USB XHCI controller and Intel IGD to a virtual machine. This is known as legacy mode passthrough, or GVT-d, where the guest is given complete control over the IGD.

The [GVT-d Setup Guide](https://github.com/intel/gvt-linux/wiki/GVTd_Setup_Guide) may be useful. For GVT-g, where virtualized GPU's are made available to guests for hardware acceleration, see [Intel GVT-g](https://wiki.archlinux.org/index.php/Intel_GVT-g).

## Requirements

1. You need to extract the IGD ROM
    1. Enable non-UEFI mode in BIOS
    2. Boot a livecd of your choice
    3. Extract ROM

```bash
echo 1 > /sys/bus/pci/devices/0000:00:02.0/rom
cat /sys/bus/pci/devices/0000:00:02.0/rom > /tmp/Intel-Iris-Plus-650.rom
echo 0 > /sys/bus/pci/devices/0000:00:02.0/rom
```

    4. Fix the ROM

```bash
$ git clone https://github.com/awilliam/rom-parser.git && cd rom-parser
$ make
gcc -o rom-parser rom-parser.c
gcc -DFIXER -o rom-fixer rom-parser.c
$ cp /tmp/Intel-Iris-Plus-650.rom ./Intel-Iris-Plus-650.rom
$ ./rom-fixer Intel-Iris-Plus-650.rom-original
Valid ROM signature found @0h, PCIR offset 40h
  PCIR: type 0 (x86 PC-AT), vendor: 8086, device: 0406, class: 030000
  PCIR: revision 3, vendor revision: 0

Modify vendor ID 8086? (y/n): n
Modify device ID 0406? (y/n): y
New device ID: 5927
Overwrite device ID with 5927? (y/n): y
  Last image
ROM checksum is invalid, fix? (y/n): y
```

    5. Copy `Intel-Iris-Plus-650.rom` to `/var/lib/libvirt/images/` of the host.

2. You need to download a Windows 10 ISO

(You can do this after installing the host)

Copy the ISO to `/var/lib/libvirt/images/` of the host.

3. You need to download stable virtio drivers

(You can do this after install the host)

`$ wget https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/archive-virtio/virtio-win-0.1.141-1/virtio-win-0.1.141.iso`

Copy the ISO to `/var/lib/libvirt/images/` of the host.

4. You need to revert to UEFI in the BIOS

## Host Configuration

### Hardware

* Intel NUC NUC7i7BNB
  * 32 GB RAM
  * 1 TB SSD

## Firmware (BIOS/UEFI)

* BIOS Configuration
  * Legacy mode (EFI mode untested)

## Software (OS)

* Ubuntu 19.04 (disco) Server (amd64)

Install `qemu`, `libvirt`, tools

```bash
sudo apt-get install -y qemu-system vim-nox libvirt-daemon-system apparmor-utils
sudo systemctl enable libvirtd
```

1. Update `grub`

2. Set the following in `/etc/default/grub`

`GRUB_CMDLINE_LINUX_DEFAULT="quiet splash kvm.ignore_msrs=1 intel_iommu=on iommu=pt"`

`$ sudo update-grub`

NOTE: Without `kvm.ignore_msrs=1`, applications which read unhandled MSRs will cause KVM guests to crash (BSOD).

3. Reboot

4. Detach `i915`

  TODO: Automate this

Running these commands will disable the screen output from the NUC. This is expected. It is recommended to proceed from here on using ssh and pubkey authentication, which almost makes remote management via `virt-manager` easier.

```bash
echo 0 > /sys/class/vtconsole/vtcon1/bind
rmmod snd_hda_intel
rmmod i915
```

If using qemu directly, you will need to manually manage the `vfio-pci` devices.

Pass the VGA device to `vfio-pci`

```bash
modprobe vfio-pci
echo "8086 5927" > /sys/bus/pci/drivers/vfio-pci/new_id
```

5. AppArmor

AppArmor does not play nicely with libvirt. TOOD: Make it play nicely. Interim:

`$ aa-complain libvirtd`

6. Create VM disk

`$ qemu-img create -f qcow2 /var/lib/libvirt/images/win10.qcow2 80G`

7. Dry run with qemu

This example runs qemu without a NIC. You may need to passthrough a USB device to send commands to the guest as well.

```bash
qemu-system-x86_64 \
    -enable-kvm \
    -m 16384 \
    -smp 2 \
    -cpu host \
    -cdrom /var/lib/libvirt/images/Win10_1809Oct_v2_English_x64.iso \
    -hda /var/lib/libvirt/images/win10.qcow2 \
    -vga none -nographic \
    -device vfio-pci,host=00:02.0,id=hostdev0,bus=pci.0,addr=0x2,rombar=1,romfile=/var/lib/libvirt/images/Intel-Iris-Plus-650.rom \
    -vnc :2 \
    -nic none
```

8. Power off

Kill the qemu process.

## Guest Configuration

### libvirt

This example sets up libvirt to run the guest.

#### virt-manager

TODO: Use `virt-install`?

1. Create a new guest OS. Set the PC type to i440, instead of the default q35 for Windows 10 guests. In virt-manager, attach the PCI device 0000:00:02. As IDE CD-ROM drives, add both ISOs. Update the boot-order to boot the Windows install ISO first. Change the hard disk "Disk 1" to SATA if it is not already. Ensure a "Controller Virtio SCSI" is present.

You might be asked to start the default network. Do so and run `virsh net-autostart default`.

`apparmor` on ubuntu may be overly cautious, causing errors such as

```bash
apparmor="DENIED" operation="open" profile="libvirt-67bd587e-b982-4813-a875-1b54e1f37e5e" name="/var/lib/libvirtd/images/Intel-Iris-Plus-650.rom"
```

Using `aa-complain libvirtd` does not fix this, nor does the longer command that includes the KVM UUID. A hacky, insecure workaround is to put the ROM file in /var/tmp, update the XML to the new path, then modify `/etc/apparmor.d/abstractions/libvirt-qemu` to allow read access to `/var/tmp` by changing `/{,var/}tmp/ r,` to `/{,var/}tmp/* r,`.

2. Once the guest is created the XML must be manually changed.

Ensure rom bar is enabled and the ROM file is known.

```xml
    <hostdev mode='subsystem' type='pci' managed='yes'>
      <driver name='vfio'/>
      <source>
        <address domain='0x0000' bus='0x00' slot='0x02' function='0x0'/>
      </source>
      <alias name='hostdev0'/>
      <rom bar='on' file='/var/lib/libvirt/images/Intel-Iris-Plus-650.rom'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x02' function='0x0'/>
    </hostdev>
```

3. Change to CPU Passthrough mode

```xml
  <cpu mode='host-passthrough' check='partial'>
    <topology sockets='1' cores='2' threads='2'/>
  </cpu>
```

4. Passthrough USB HID devices or setup USB passthrough

In virt-manager, you will either need to passthrough the USB keyboard and mouse or the USB PCI devices themselves.

Attaching the keyboard and mouse are easy using virt-manager.

It is recommended to perform USB passthrough during the first setup phase. See below for details on passing through the PCI devices themselves.

4. Install Windows 10

Install Windows 10. Install the `qemu-guest-agent` from the virtio ISO. Install drivers for all devices not bound to one in Device Manager.

Note: The Intel IGD is initially recognized as a "Microsoft Basic Display Adapter" device. Manually updating the device driver should make it seen as "Intel(R) Iris(R) Plus Graphics 650". If host reboots do not work, perform all Windows updates. Pray.

5. Update hardware configuration

Attach the following PCI devices in virt-manager:

```xml
    <hostdev mode='subsystem' type='pci' managed='yes'>
      <driver name='vfio'/>
      <source>
        <address domain='0x0000' bus='0x00' slot='0x14' function='0x0'/>
      </source>
      <alias name='hostdev1'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x0a' function='0x0'/>
    </hostdev>
    <hostdev mode='subsystem' type='pci' managed='yes'>
      <driver name='vfio'/>
      <source>
        <address domain='0x0000' bus='0x00' slot='0x14' function='0x2'/>
      </source>
      <alias name='hostdev2'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x0b' function='0x0'/>
    </hostdev>
```

Modify the Disk 1 disk bus from SATA to SCSI. Enable discard and "detect zeroes" (options under "Performance Options" for recent virt-manager relases.)

Remove the CD-ROMs altogether.

## Features

* Windows can be started/shutdown/started and rebooted without issue.
* The VM can be managed quite easily via ssh+qemu using virt-manager.

## IOMMU Groups

IOMMU groups are based on the hardware architecture of the system. In the case of the NUC7i7BNH, most devices are either isolated or behind the Low Pin Count (LPC) Controller at `1f.0`. In these examples, we explicilty passthrough the Iris Plus Graphics 650 at `02.0` and the USB controller at `16.0`. 

```bash
$ lspci -t -nn -v
-[0000:00]-+-00.0  Intel Corporation Xeon E3-1200 v6/7th Gen Core Processor Host Bridge/DRAM Registers [8086:5904]
           +-02.0  Intel Corporation Iris Plus Graphics 650 [8086:5927]
           +-08.0  Intel Corporation Xeon E3-1200 v5/v6 / E3-1500 v5 / 6th/7th Gen Core Processor Gaussian Mixture Model [8086:1911]
           +-14.0  Intel Corporation Sunrise Point-LP USB 3.0 xHCI Controller [8086:9d2f]
           +-14.2  Intel Corporation Sunrise Point-LP Thermal subsystem [8086:9d31]
           +-16.0  Intel Corporation Sunrise Point-LP CSME HECI #1 [8086:9d3a]
           +-17.0  Intel Corporation Sunrise Point-LP SATA Controller [AHCI mode] [8086:9d03]
           +-1c.0-[01-39]--
           +-1c.5-[3a]----00.0  Intel Corporation Wireless 8265 / 8275 [8086:24fd]
           +-1c.7-[3b]----00.0  Realtek Semiconductor Co., Ltd. RTS5229 PCI Express Card Reader [10ec:5229]
           +-1f.0  Intel Corporation Intel(R) 100 Series Chipset Family LPC Controller/eSPI Controller - 9D4E [8086:9d4e]
           +-1f.2  Intel Corporation Sunrise Point-LP PMC [8086:9d21]
           +-1f.3  Intel Corporation Sunrise Point-LP HD Audio [8086:9d71]
           +-1f.4  Intel Corporation Sunrise Point-LP SMBus [8086:9d23]
           \-1f.6  Intel Corporation Ethernet Connection (4) I219-V [8086:15d8]
```

```bash
for d in /sys/kernel/iommu_groups/*/devices/*; do n=${d#*/iommu_groups/*}; n=${n%%/*}; printf 'IOMMU Group %s ' "$n"; lspci -nns "${d##*/}"; done;
```

## NUC7i7BNH

```text
IOMMU Group 0 00:00.0 Host bridge [0600]: Intel Corporation Xeon E3-1200 v6/7th Gen Core Processor Host Bridge/DRAM Registers [8086:5904] (rev 03)
IOMMU Group 1 00:02.0 VGA compatible controller [0300]: Intel Corporation Iris Plus Graphics 650 [8086:5927] (rev 06)
IOMMU Group 2 00:08.0 System peripheral [0880]: Intel Corporation Xeon E3-1200 v5/v6 / E3-1500 v5 / 6th/7th Gen Core Processor Gaussian Mixture Model [8086:1911]
IOMMU Group 3 00:14.0 USB controller [0c03]: Intel Corporation Sunrise Point-LP USB 3.0 xHCI Controller [8086:9d2f] (rev 21)
IOMMU Group 3 00:14.2 Signal processing controller [1180]: Intel Corporation Sunrise Point-LP Thermal subsystem [8086:9d31] (rev 21)
IOMMU Group 4 00:16.0 Communication controller [0780]: Intel Corporation Sunrise Point-LP CSME HECI #1 [8086:9d3a] (rev 21)
IOMMU Group 5 00:17.0 SATA controller [0106]: Intel Corporation Sunrise Point-LP SATA Controller [AHCI mode] [8086:9d03] (rev 21)
IOMMU Group 6 00:1c.0 PCI bridge [0604]: Intel Corporation Sunrise Point-LP PCI Express Root Port #1 [8086:9d10] (rev f1)
IOMMU Group 7 00:1c.5 PCI bridge [0604]: Intel Corporation Sunrise Point-LP PCI Express Root Port #6 [8086:9d15] (rev f1)
IOMMU Group 8 00:1c.7 PCI bridge [0604]: Intel Corporation Sunrise Point-LP PCI Express Root Port #8 [8086:9d17] (rev f1)
IOMMU Group 9 00:1f.0 ISA bridge [0601]: Intel Corporation Intel(R) 100 Series Chipset Family LPC Controller/eSPI Controller - 9D4E [8086:9d4e] (rev 21)
IOMMU Group 9 00:1f.2 Memory controller [0580]: Intel Corporation Sunrise Point-LP PMC [8086:9d21] (rev 21)
IOMMU Group 9 00:1f.3 Audio device [0403]: Intel Corporation Sunrise Point-LP HD Audio [8086:9d71] (rev 21)
IOMMU Group 9 00:1f.4 SMBus [0c05]: Intel Corporation Sunrise Point-LP SMBus [8086:9d23] (rev 21)
IOMMU Group 9 00:1f.6 Ethernet controller [0200]: Intel Corporation Ethernet Connection (4) I219-V [8086:15d8] (rev 21)
IOMMU Group 10 3a:00.0 Network controller [0280]: Intel Corporation Wireless 8265 / 8275 [8086:24fd] (rev 78)
IOMMU Group 11 3b:00.0 Unassigned class [ff00]: Realtek Semiconductor Co., Ltd. RTS5229 PCI Express Card Reader [10ec:5229] (rev 01)
```

Note that many of the devices are in their own IOMMU groups, making it easier to perform passthrough. This arrangement will change based on the hardware and the BIOS configuration.