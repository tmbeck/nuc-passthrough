# PXE Booting in KVM

## i440, BIOS, PXE boot

File download order example:

```text
pxelinux.0
pxelinux.cfg/8749e0c8-e1d3-6541-9a5f-dd1ca48e2f63
pxelinux.cfg/01-f6-dd-7a-86-ce-62
pxelinux.cfg/C0A80269
pxelinux.cfg/C0A8026
pxelinux.cfg/C0A802
pxelinux.cfg/C0A80
pxelinux.cfg/C0A8
pxelinux.cfg/C0A
pxelinux.cfg/C0
pxelinux.cfg/C
pxelinux.cfg/default
menu.c32
pxelinux.cfg/default
```

Key files:

* `pxelinux.0` the PXE Linux boot loader
* `menu.c32` the menu system binary
* `pxelinux.cfg/default` the menu layout and configuration

Example of the `default` configuration file for booting MemTest86+

```bash
default menu.c32
prompt 1
timeout 30
MENU TITLE PXE Menu

###################
## Non-UEFI only ##
###################

LABEL memtest
  MENU LABEL MemTest86+ 5.01
  MENU default
  KERNEL /path/to/memtest86+-5.01
```

In this example, `/path/to/memtest86+-5.01` is relative to the root of the tftpd binary, typically `/var/lib/tftpboot`.

## q35, OVMF (UEFI), PXE Boot

File download order example:

```text
uefi/shimx64.efi
uefi/shimx64.efi
uefi/grubx64.efi
/uefi/grub.cfg-01-06-d5-e1-3c-2f-75
/uefi/grub.cfg-C0A803E1
/uefi/grub.cfg-C0A803E
/uefi/grub.cfg-C0A803
/uefi/grub.cfg-C0A80
/uefi/grub.cfg-C0A8
/uefi/grub.cfg-C0A
/uefi/grub.cfg-C0
/uefi/grub.cfg-C
/uefi/grub.cfg
/EFI/centos/grub.cfg-01-06-d5-e1-3c-2f-75
/EFI/centos/grub.cfg-C0A803E1
/EFI/centos/grub.cfg-C0A803E
/EFI/centos/grub.cfg-C0A803
/EFI/centos/grub.cfg-C0A80
/EFI/centos/grub.cfg-C0A8
/EFI/centos/grub.cfg-C0A
/EFI/centos/grub.cfg-C0
/EFI/centos/grub.cfg-C
/EFI/centos/grub.cfg
/EFI/centos/x86_64-efi/command.lst
/EFI/centos/x86_64-efi/fs.lst
/EFI/centos/x86_64-efi/crypto.lst
/EFI/centos/x86_64-efi/terminal.lst
/EFI/centos/grub.cfg
```

