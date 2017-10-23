# unRAID bzoverlay

## Big fat disclaimer

Modifying your unRAID server in **any** way may always put your data at risk. This, of course, also applies to this modification. You should always keep backups of any files you absolutely can not afford to lose. **Any and all responibility for problems that may arise as a result from modifying unRAID in any way is yours, and yours alone!**

## Run unRAID from a hard drive - the easy way.

I run an unRAID server in my home network since a few years, and I absolutely love the product.

unRAID boots a root filesystem into ram from a usb flash drive, matches the GUID on the flash drive against the license key and uses the flash drive for persistent storage. Simple and elegant!

If it weren't for the fact that usb flash drives tend to go bad, I would happily run unRAID from a usb flash drive forever.

However, having had two flash drives go bad, I started to look into keeping the usb flash only for the GUID and boot the system from hard drive. Limetech (the makers of unRAID) do have an excellent key replacement policy and has provided me with new license keys, but downtime is downtime.

The ultimate goal was to keep unRAID working **exactly** as usual, only from a hard drive instead.

As it turns out, I am not the first person to entertain this idea - and I found one complicated way after another. Some required that the root filesystem needs to be unpacked, some bootup scripts rewritten and repacked. Some still boots from the usb flash and shuffles the mounts around after boot.

While I am perfectly comfortable with patching initramfs, it is still and inconvenience and it needs to be done after every new version of unRAID.

Some even installed the unRAID components over a Slackware system, compiled a new kernel with Limetechs patches and ran everything straight from the disk.

None of these options are acceptable for something that should **just work™**.


Here are some of my findings while researching how to boot from hard drive, while still keeping full functionality and convenience:

* The flash drive needs to be fat32 (and possibly ntfs?) and must be labeled 'UNRAID'. It also needs to be mounted somewhere for the licensing to work.
* The hard drive must boot from a fat32 partition, and can not be labelled UNRAID.
* I entertained the idea of booting from a btrfs subvolume, but the kernel panics at an early stage if trying to mount btrfs. Since this isn't really a deal breaker, I didn't look into it any further, so it may still be possible.
* The system mounts any drive labeled UNRAID as read/write in the location for persistent storage at boot.
* Trying to trick unRAID into mounting another drive by relabelling them, will not work since it will not find the GUID needed for the license if the flash drive has the wrong label.

The solution is as simple as it is elegant;

* Create a tiny overlay initrafs which mounts the flash drive read-only somewhere and mounts the harddrive as persistent storage. All we really need is a new /etc/fstab with different mount moints and a /license directory to mount the key.
* Add the overlay to the list of initramfs files to load at boot.

Thats it! No unpacking and patching the original files. No moving mountpoints after boot. Nada.

This allows unRAID to find the GUID needed for license, as well as using the hard drive right from the start without having to shuffle anything around after boot.

And, since we're only using a tiny overlay, any updates to the base image should (in theory) **just work™**. We are not really changing how unRAID expects things to work.

##### This technique will work equally well when installing unRAID as a guest OS where boot from usb may not even be an option. No need to mount a plop image to chainboot into usb flash - just create a disk image with these modifications, add the usb key to the vm and off you go! :)

## Manual installation

#### This needs to be done only once!

You can do this right on the unRAID server. If you are uncomfortable with that, just hook the hard drive and usb flash drive up to another Linux computer to do the installation. You will need root privileges for (most of) the following commands. Sudo into a root shell or just add sudo to each line as needed.

##### In the following instruction I am installing unRAID to /dev/sdx1. You are solely responsible for finding out which drive and partition you will install to! Formatting the wrong partition will destroy data!

Create a boot partition on your hard drive. If installing syslinux by hand, it can be any primary partition. If following the instructions below, use the first partition on the disk.

Create filesystem and mount it
```
mkfs.vfat -n BOOTDISK /dev/sdx1
mkdir /mnt/unraid_disk
mount /dev/sdx1 /mnt/unraid_disk
```

Copy unRAID files from usb flash drive. (Substitute /boot/* with path to usb flash drive, if not on unraid system.)
```
cp -R /boot/* /mnt/unraid_disk
```

Patch make_bootable_linux and execute it to install syslinux to the hard drive.
```
sed -e "s|UNRAID|BOOTDISK|" -i /mnt/unraid_disk/make_bootable_linux
/mnt/unraid_disk/make_bootable_linux
```

Alt 1: Create the initramfs overlay by hand (need cpio installed)
```
mkdir -p rootfs/{etc,license}
cat << EOF > rootfs/etc/fstab
/dev/disk/by-label/UNRAID   /license vfat auto,ro,shortname=mixed 0 1
/dev/disk/by-label/BOOTDISK /boot    vfat auto,rw,exec,noatime,nodiratime,umask=0,shortname=mixed 0 1
EOF
cd rootfs
find . | cpio -o -H newc | xz > /mnt/unraid_disk/bzoverlay
```

Alt 2: Install my premade overlay (Does not need cpio)
```
wget https://raw.githubusercontent.com/thohell/unRAID-bzoverlay/master/bzoverlay -O /mnt/unraid_disk/bzoverlay
```


Patch syslinux.cfg to load our overlay at boot
```
sed -e "s|/bzroot|/bzroot,/bzoverlay|" -i /mnt/unraid_disk/syslinux/syslinux.cfg

```

#### Done! Change boot device in BIOS and start up your unRAID server. Everything should work EXACTLY as it did before and upgrades should work exactly as usual, but the flash drive will never get written to again. This should make it less likely to go bad.


