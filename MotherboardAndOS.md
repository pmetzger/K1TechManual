# Motherboard

The main motherboard runs
[Linux](https://en.wikipedia.org/wiki/Linux) on a
[MIPS](https://en.wikipedia.org/wiki/MIPS_architecture) based
processor. The CPU itself is an
[SoC](https://en.wikipedia.org/wiki/System_on_a_chip) from a Chinese
fabless semiconductor company called
[Ingenic](https://en.wikipedia.org/wiki/Ingenic_Semiconductor).

The SoC itself is Ingenic's
[X2000E](http://www.ingenic.com.cn/en/?product/id/21.html).

This is a dual MIPS 32 bit processor at 1.2GHz with 256MB of RAM. The
machine also has an [eMMC
module](https://en.wikipedia.org/wiki/MultiMediaCard#eMMC) with about
7.2GB of storage. (Note: this size is based on the last sector of the
eMMC device's [GUID partition table
(GPT)](https://en.wikipedia.org/wiki/GUID_Partition_Table) as printed
by the [fdisk](https://man7.org/linux/man-pages/man8/fdisk.8.html)
command, and not on running
[lsblk](https://linux.die.net/man/8/lsblk), which is not available on
the default firmware.)

# Operating System

The main motherboard runs Linux. The system image is based on a
[board support package (BSP)](https://en.wikipedia.org/wiki/Board_support_package) from
Ingenic itself that may be downloaded from their
[ftp site](ftp://ftp.ingenic.com.cn/).

The BSP development package builds its system images using an Ingenic
fork of [buildroot](https://buildroot.org/), a commonly used system
for constructing embedded Linux images.

The firmware image relies on
[BusyBox](https://en.wikipedia.org/wiki/BusyBox) for many of the
executables including the init system and main system daemons.  Other
executables have been added onto the image by Creality's developers.

Some documentation on the BusyBox tools may be found in the
[manual page](https://man.archlinux.org/man/busybox.1.en). 

The init system is of the sysv init.d variety but lacks runlevels and
most other features. The tasks that are started at boot are split
between `/etc/inittab` and a set of start scripts located in
`/etc/init.d`. It is believed that system boot is relatively slow
because of the architecture of the init system, and that it could be
significantly sped up by using an alternative. The init system does
have the advantage of being simple and reliable.

The partition table for the eMMC is as follows:

```
Number  Start (sector)    End (sector)  Size Name
     1            2048            4095 1024K ota
     2            4096            6143 1024K sn_mac
     3            6144           14335 4096K rtos
     4           14336           22527 4096K rtos2
     5           22528           38911 8192K kernel
     6           38912           55295 8192K kernel2
     7           55296          669695  300M rootfs
     8          669696         1284095  300M rootfs2
     9         1284096         1488895  100M rootfs_data
    10         1488896        15271935 6730M userdata
```

You will notice that several partitions are duplicated, with `name`
and `name2` variants. Only one of the two is active at any given
time. When firmware is upgraded, the inactive set of partitions have
new firmware placed in them, and after the firmware is completely in
place, which partition is active is switched by writing either
`ota:kernel` or `ota:kernel2` into the `ota` partition (known to Linux
as `/dev/mmcblk0p1`. You can see which is active simply by doing:

```
[#] cat /dev/mmcblk0p1
ota:kernel2
```

which will show which set of partitions are currently active and will
be used at next reboot.

(One wonders why these aren't "rtos1/rtos2" or "rtos0/rtos1" or what
have you.)


Similarly, the machine's serial number, model, MAC address, etc. may be
found by looking at the `sn_mac` partition by doing `cat /dev/mmcblk0p2`.

Running [mount](https://man7.org/linux/man-pages/man8/mount.8.html) on
the running system will reveal a number of file systems.

```
[#] mount
/dev/root on /rom type squashfs (ro,relatime)
devtmpfs on /dev type devtmpfs (rw,relatime,size=100748k,nr_inodes=25187,mode=755)
proc on /proc type proc (rw,relatime)
devpts on /dev/pts type devpts (rw,relatime,gid=5,mode=620,ptmxmode=666)
tmpfs on /dev/shm type tmpfs (rw,relatime,mode=777)
tmpfs on /tmp type tmpfs (rw,relatime)
tmpfs on /run type tmpfs (rw,nosuid,nodev,relatime,mode=755)
sysfs on /sys type sysfs (rw,relatime)
debugfs on /sys/kernel/debug type debugfs (rw,relatime)
/dev/mmcblk0p9 on /overlay type ext4 (rw,sync,relatime,block_validity,delalloc,barrier,user_xattr)
overlayfs:/overlay on / type overlay (rw,sync,noatime,lowerdir=/,upperdir=/overlay/upper,workdir=/overlay/work)
/dev/mmcblk0p10 on /usr/data type ext4 (rw,sync,relatime,block_validity,delalloc,barrier,user_xattr)
```

Note that the root device isn't listed by this, but

```
[#] cat /proc/cmdline
console=ttyS2,115200n8  mem=256M@0x0 mem=0M@0x30000000 lcm_id=3  init=/linuxrc root=/dev/mmcblk0p8 rootwait rootfstype=squashfs ro
```

reveals it is a
[squashfs](https://docs.kernel.org/filesystems/squashfs.html) located
on either `/dev/mmcblk0p7` ("kernel") or `dev/mmcblk0p7`
("kernel2"). So, `/rom` is not really a ROM but rather is a read-only
partition in eMMC.

In order to allow writing of configuration files and the like, an
ext4fs file system is mounted on `/overlay` and then overlay mounted
over `/`; files that are "overlayed" over the main file system are
then found in `/overlay/work`. The overlay file system is located on
partition 9, a.k.a. `/dev/mmcblk0p9`, a.k.a. `rootfs_data`.

Larger files (including things like STLs the user uploads, much of the
[Klipper](https://www.klipper3d.org/) configuration, new firmware that
has just been downloaded by the over the air update system, etc.) is
located in /usr/data, a 6.5GB ext4fs partition located on partition
10, a.k.a. `/dev/mmcblk0p10`, a.k.a. `userdata`.  (It is kind of a
catch-all location for extra stuff that needs more room; the overlay
file system is quite small at 100MB and is not large enough for such
things.)

Much of the usual `/var` directory is symlinked to ramdisk filesystems
mounted on `/tmp` and `/run`. This includes things like `/var/log`, so
logs from that location do not survive reboots.


The root file system image is built using a hacked version of
https://buildroot.org/ — Ingenic has their fork of the buildroot stuff
available for download.


The current system runs a Linux 4.4.94 kernel; it might be nice
to have a newer, less buggy kernel, as the current one has problems
like occasional hangs of the WiFi hardware.

# Booting

The system boot firmware is
[uboot](https://en.wikipedia.org/wiki/Das_U-Boot), an open source,
GPLed boot loader. During boot, system messages are sent to the
motherboard's serial console. The serial console is available on three
logic level pins on the motherboard, and runs at 115200bps, 8 bits, no
parity, one stop bit. It can be easily connected to with an
inexpensive USB to serial pin converter. (The system touch screen does
not seem to be associated with a tty of any sort.)

# Firmware updates

Firmware updates get the extension `.img`, and consist of (encrypted,
though it's not overly hard to figure out the key) 7z archive files
that contain sliced up parts of the root file system squashfs,
the kernel, and the `rtos` partition. (The present author is not 100%
clear on the function of the `rtos` partition; it is presumably
firmware of some sort.)

You can simply put a firmware update file in the root partition of FAT
filesystem on a USB key into the front of the device, and if it has
the right name, the system will notice it, ask if it should load it,
and if you say yes, it will update itself. The system also polls
Creality to find updates and download them off the internet, which it
will also offer for update.

The version of the firmware that was last loaded is described in the
file `/etc/ota_info`. The update process itself is conducted by
a group of shell scripts located in `/etc/ota_bin/`; they are not very
difficult to read but have some bugs in them. (If Creality would like
bug reports, please get in touch.)

If the update process bricks the machine, it is possible to load
arbitrary data onto the eMMC module over a micro USB connector located
on the motherboard. Given appropriate software and an upload image,
one can always recover a machine that has been bricked because of
corrupted firmware.

# Additional notes

Memory in the environment is very constrained, which may be why some
of the OpenCV based AI camera etc. features aren't working yet. Only a
bit under 70 meg of RAM are free on the test machine being used for
writing this documentation. Creality has indicated on Github there are
some problems with [Moonraker](https://github.com/Arksine/moonraker)
installations running out of memory, which may be caused by
this. (Although 256MB was once a great deal of RAM, it is now very
small, and even a Rapberry Pi Zero has twice as much.) This may be a
good reason for uploading gcode files and remote printing them rather
than streaming prints to the machine.

There seem to be bugs in current firmware with WiFi. The system log is
showing disconnects from the WiFi once in a while, probably because
the (relatively old) Linux kernel has some bugs that are fixed in
later releases.

[SSH](https://en.wikipedia.org/wiki/Secure_Shell) service is provided
by [dropbear](https://en.wikipedia.org/wiki/Dropbear_(software)),
which reasons the present author doesn't understand has been compiled
specifically so it will only accept obsolete DSA keys. If you set your
ssh client up to use a dsa key, you can do public key auth to the
machine instead of password based.

# Is this a good enough Motherboard?

One can see why Creality would chose to use something like this
Ingenic SoC to put Klipper on their lower end printers where the level
of integration and low cost probably made it possible to Klipperize
the machines at a very low price point. However, such hardware is
somewhat disappointing on higher end machines where the memory and CPU
speed limitations may make it hard to add some new features as
software improves. One hopes future hardware from Creality has a bit
more headroom, and even that they make newer controller boards
available for the K1 series for people who want to try newer control
software ideas that might require a bit more room.

In the meanwhile, it may be nice for people to experiment with the use
of better main motherboards, provided the software can be kept
compatible enough.
