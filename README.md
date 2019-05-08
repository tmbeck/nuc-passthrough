# Intel NUC NUC7i7BNH IGD Passthrough

## Requirements

1. You need to extract the IGD ROM
    1. Enable non-UEFI mode in BIOS
    2. Boot a livecd of your choice
    3. Extract ROM

```
$ echo 1 > /sys/bus/pci/devices/0000:00:02.0/rom
$ cat /sys/bus/pci/devices/0000:00:02.0/rom > /tmp/Intel-IGD-Iris-650.rom
$ echo 0 > /sys/bus/pci/devices/0000:00:02.0/rom
```

Copy `Intel-IGD-Iris-650.rom` to `/var/lib/libvirt/images/` of the host.

2. You need to download a Windows 10 ISO

(You can do this after installing the host)

Copy the ISO to `/var/lib/libvirt/images/` of the host.

3. You need to download stable virtio drivers

(You can do this after install the host)

`$ wget https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/archive-virtio/virtio-win-0.1.141-1/virtio-win-0.1.141.iso`

Copy the ISO to `/var/lib/libvirt/images/` of the host.

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

```
$ sudo apt-get install -y qemu-system vim-nox libvirt-daemon-system
$ sudo systemctl enable libvirtd
```

1. Update `grub`

2. Set the following in `/etc/default/grub`

`GRUB_CMDLINE_LINUX_DEFAULT="quiet splash kvm.ignore_msrs=1 intel_iommu=on iommu=pt"
`

`$ sudo update-grub`

3. Reboot

4. Detach `i915`

  TODO: Automate this

Running these commands will disable the screen output from the NUC. This is expected. It is recommended to proceed from here on using ssh and pubkey authentication, which almost makes remote management via `virt-manager` easier.

```
$ echo 0 > /sys/class/vtconsole/vtcon1/bind
$ rmmod snd_hda_intel
$ rmmod i915
```

If using qemu directly, you will need to manually manage the `vfio-pci` devices.

Pass the VGA device to `vfio-pci`

```
modprobe vfio-pci
echo "8086 5927" > /sys/bus/pci/drivers/vfio-pci/new_id
```

5. AppArmor

AppArmor does not play nicely with libvirt. TOOD: Make it play nicely. Interim:

` aa-complain libvirtd`

6. Create VM disk

`$ qemu-img create -f qcow2 /var/lib/libvirt/images/win10.qcow2 80G`

7. Dry run with qemu

This example runs qemu without a NIC. You may need to passthrough a USB device to send commands to the guest as well.

```
qemu-system-x86_64 \
    -enable-kvm \
    -m 16384 \
    -smp 2 \
    -cpu host \
    -cdrom /var/lib/libvirt/images/Win10_1809Oct_v2_English_x64.iso \
    -hda /var/lib/libvirt/images/win10.qcow2 \
    -vga none -nographic \
    -device vfio-pci,host=00:02.0,id=hostdev0,bus=pci.0,addr=0x2,rombar=1,romfile=/var/lib/libvirt/images/Intel-IGD-Iris-650.rom \
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

2. Once the guest is created the XML must be manually changed.

Ensure rom bar is enabled and the ROM file is known.

```xml
    <hostdev mode='subsystem' type='pci' managed='yes'>
      <driver name='vfio'/>
      <source>
        <address domain='0x0000' bus='0x00' slot='0x02' function='0x0'/>
      </source>
      <alias name='hostdev0'/>
      <rom bar='on' file='/var/lib/libvirt/images/Intel-IGD-Iris-650.rom'/>
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

Install Windows 10. Install the qemu-guest-agent from the virtio ISO. Install drivers for all devices not bound to one in Device Manager.

Note: It took a few reboots for the Intel IGD to be properly recognized (it was a "Basic" display device at first). If host reboots do not work, perform all Windows updates.

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

Modify the Disk 1 disk bus from SATA to SCSI.

Remove the CD-ROMs altogether.
