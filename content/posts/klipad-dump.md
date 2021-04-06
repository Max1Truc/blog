---
title: "Dumping user data from cheap tablets in minutes"
date: 2021-04-06
draft: false
---

Hey!
So I recently found a Klipad, which was waiting somewhere in my house.
It is a cheap, tiny tablet, and I wanted to tinker a bit with it.
I then decided to somewhat root it, or even install *GNU/* Linux on it.
As it is running Android 8.1.0 and didn't receive any security patches since something like September 2019, exploit-db.com gave me some ideas of [root exploits].
However, that would not be the simplest way to install *GNU/* Linux, if it is even possible without breaking everything.

## Trying stuff

I went into the recovery mode (using `adb reboot recovery` from my pc), hoping it doesn't require a signature when flashing updates, and it did require it. Signatures are a way for the device to verify that the updates it gets are from its manufacturer. I could not create DIY updates to do anything I wanted.

While trying some other random things, I executed `adb reboot bootloader`, which usually reboots the phone in a weird way, and gives access to something different from normal Android, and the recovery. I supposed it would give me some kind of fastboot mode or odin mode, which are other ways to send updates on most phones. I would maybe have the capability to unlock the tablet, and upload my own updates.

However, all I saw was a black screen. As it was supposed to be some kind of reboot, and not a shutdown, I wired it to my Linux PC, and ran `lsusb` to see if I could talk with it:
```bash
$ lsusb
Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
[...]
Bus 001 Device 018: ID 2207:310d Fuzhou Rockchip Electronics Company RK3126 in Mask ROM mode
[...]
Bus 003 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
```

ROM is the memory a device, and stand for Read-Only Memory, but it is usually writable anyway, so it is often used to refer to the Android software which runs on the device. "Stock ROM" refers to the official software created by the manufacturer of the device, while "Custom ROM" refers to a ROM created by anyone else. LineageOS and GrapheneOS are examples of custom ROMs for Android phones.
So the device is in `Mask ROM mode`, may I upload my own custom ROM ?

After copy-pasting the name of the device on Qwant, I got on a wiki by [kobol.io]. *Sroll-scroll-scroll* And... it mentions some commands starting with `sudo tools/rkdeveloptool`.

Let's search rkdevelop on the AUR (Arch User Repository, a land of software) !
```bash
$ yay -Ss rkdevelop
aur/rkdeveloptool 66-1 (+2 0.01)
    Development tool for Rockchip SOC
```

So, I can simply install it with `yay -S rkdeveloptool`. Let's see what it can do.

```bash
$ sudo rkdeveloptool

---------------------Tool Usage ---------------------
Help:			-h or --help
Version:		-v or --version
ListDevice:		ld
DownloadBoot:		db <Loader>
UpgradeLoader:		ul <Loader>
ReadLBA:		rl  <BeginSec> <SectorLen> <File>
WriteLBA:		wl  <BeginSec> <File>
WriteLBA:		wlx  <PartitionName> <File>
WriteGPT:		gpt <gpt partition table>
WriteParameter:		prm <parameter>
PrintPartition:		ppt
EraseFlash:		ef
TestDevice:		td
ResetDevice:		rd [subcode]
ReadFlashID:		rid
ReadFlashInfo:		rfi
ReadChipInfo:		rci
ReadCapability:		rcb
PackBootLoader:		pack
UnpackBootLoader:	unpack <boot loader>
TagSPL:			tagspl <tag> <U-Boot SPL>
-------------------------------------------------------

$ sudo rkdeveloptool ld
DevNo=1	Vid=0x2207,Pid=0x310d,LocationID=103	Loader
```

Cool, this tool recognizes my tabet.
Wait... what are these `ReadFlashInfo` and `ReadLBA` things ?
Let's try `ReadFlashInfo` first:
```bash
$ sudo rkdeveloptool rfi
Flash Info:
	Manufacturer: MICRON, value=04
	Flash Size: 16384 MB
	Flash Size: 33554432 Sectors
	Block Size: 8192 KB
	Page Size: 16 KB
	ECC Bits: 60
	Access Time: 32
	Flash CS: Flash<0>
```

Hmmmm... 33554432 sectors (~= 16 Gib)... didn't `ReadLBA` need some sector data ?
```
ReadLBA:		rl  <BeginSec> <SectorLen> <File>
```

Let's try `ReadLBA` then !
```bash
$ time sudo rkdeveloptool rl 0 33554432 image.img
Read LBA to file (100%)
sudo rkdeveloptool rl 0 33554432 image.img  8,95s user 18,99s system 3% cpu 14:40,88 total
```

So, approximately 15 minutes to dump 16 GiB, and that's a 16GiB-tablet.
Is it that simple to dump user data ?

## How can I read this data ?

```bash
$ fdisk -l image.img
Disk image.img: 16 GiB, 17179869184 bytes, 33554432 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```

This doesn't look like a normal hard disk or usb drive...
There should be some partitions like "system" or "userdata" or "boot" but it shows nothing.

Let's manually look at it !
```bash
$ dd if=image.img count=2 2>/dev/null | cat
PARM?FIRMWARE_VER:7.1
MACHINE_MODEL:RK3126c
MACHINE_ID:007
MANUFACTURER:rk3126c
MAGIC: 0x5041524B
ATAG: 0x00200800
MACHINE: 3126c
CHECK_MASK: 0x80
PWR_HLD: 0,0,A,0,1
CMDLINE: console=ttyFIQ0 androidboot.baseband=N/A androidboot.veritymode=enforcing androidboot.hardware=rk30board androidboot.console=ttyFIQ0 init=/init initrd=0x62000000,0x00800000 mtdparts=rk29xxnand:0x00002000@0x00002000(uboot),0x00002000@0x00004000(trust),0x00002000@0x00006000(misc),0x00008000@0x00008000(resource),0x00010000@0x00010000(kernel),0x00010000@0x00020000(boot),0x00020000@0x00030000(recovery),0x00038000@0x00050000(backup),0x00002000@0x00088000(security),0x00100000@0x0008a000(cache),0x00280000@0x0018a000(system),0x00008000@0x0040a000(metadata),0x00038000@0x00412000(vendor),0x00008000@0x0044a000(oem),0x00000400@0x00452000(frp),-@0x00452400(userdata)
�XX
```

I don't really know what kind of magic (header) `PARM` is.
However, CMDLINE looks like Linux Kernel parameters !
Also, I actually see some `boot`, `system` and `userdata` !
After some [digging on rockchip linux kernel], I could understand this `mtdparams=` thing.
So, let's say I want to dump `boot`.
It is from sector 0x00020000 to sector 0x00020000 + 0x00010000.
Now, I want to dump my `userdata`.
It starts at sector 0x00452400, and, as the offset is a dash (`-`), it is until the end.

## Browsing userdata

As I am on Linux, that's rather simple to mount it as a normal drive.

```bash
$ dd if=image.img skip=$((0x00452400)) | file -
/dev/stdin: F2FS filesystem, UUID=adcf4214-0553-4465-944c-bce0017cce5c, volume name ""
$ sudo mount -r -t f2fs -o loop,offset=$((0x00452400 * 512)) image.img /mnt/
$ ls /mnt
adb
anr
app
app-asec
app-ephemeral
app-lib
app-private
backup
bootchart
cache
camera
cifsmanager
dalvik-cache
data
drm
gps
local
logs
lost+found
media
[...]
```

I did create a file `hello.txt` from Termux on this tablet so let's read it:
```bash
$ sudo cat /mnt/data/com.termux/files/home/hello.txt
Hello, World !
```

This way, anyone should be able to dump my data.

## Final thoughts

This tablet is capable of user data encryption. However, using `rkdeveloptool`, anyone could change the system binary so that it saves your password in a file. Moreover, if you set a schema, it can easily be cracked, using [aplc] for example.
I think I'm going to shred all the data I can on this tablet, and, if you've got a similar one, you should too.

Next step will be to let *GNU/* Linux run on this tablet !
*postmarketOS vibes*

[root exploits]: https://hernan.de/blog/tailoring-cve-2019-2215-to-achieve-root/
[kobol.io]: https://wiki.kobol.io/helios64/maskrom/
[digging on rockchip linux kernel]: https://github.com/rockchip-linux/kernel/blob/82c9666cb6fe999eb61f23c2c9d0d5dad7332fb6/drivers/mtd/cmdlinepart.c#L23
[aplc]: https://github.com/sch3m4/androidpatternlock

