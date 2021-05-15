# Working with PC98 disks in Linux

## Mounting Images - Introduction
To mount images, we need to work with loop devices, which involves `losetup` and `mount`, which involves superuser permissions. Assuming you're working on a personal computer and not a multi-user server, this probably isn't an issue, but further assuming that you're not running as a superuser at all times, we'll need to set some options to prevent annoying permissions issues with write access and unmounting.

If you plan to make working with disk images a relatively common occurrence, it would be best to add these mount options as an fstab entry to save time:

    /dev/loop0  /mnt/loop0  auto  users,rw,exec,nofail,noauto,sync,check=strict,codepage=932,iocharset=euc-jp,x-gvfs-show  0 0

You can, of course, change the mountpoint to whatever you'd like. The important bit here is the options. `users`, `rw`, `exec`, `nofail` and `noauto` are all pretty standard and you'll likely already recognize them. The rest are important for our use case.

First `sync` will ensure that any writes happen immediately instead of being cached. This slows things down, but since we're working with old hardware and compartively tiny data, it should be negligible.

Next is `check=strict` which prevents long filenames and DOS invalid characters from being used when created filenames. (The other available options are `relaxed` and `normal` which will allow invalid characters but will convert them to be permissable in the actual file system. For this reason, `strict` is suggested so you know exactly what filename to expect.)

`codepage=932` and `iocharset=euc-jp` set the encoding/charset of the filesystem to Japanese for compatibility.

Finally, `x-gvfs-show` is optional. It will add the loop device to your file manager if you are using gvfs.

If you plan on working with multiple images at a time, you can create multiple entries:

    /dev/loop1  /mnt/loop1  auto  users,rw,exec,nofail,noauto,sync,check=strict,codepage=932,iocharset=euc-jp,x-gvfs-show  0 0
    /dev/loop2  /mnt/loop2  auto  users,rw,exec,nofail,noauto,sync,check=strict,codepage=932,iocharset=euc-jp,x-gvfs-show  0 0
    /dev/loop3  /mnt/loop3  auto  users,rw,exec,nofail,noauto,sync,check=strict,codepage=932,iocharset=euc-jp,x-gvfs-show  0 0

... and so on.

The process of accessing a disk image is two-fold: attach the file to a loop device and mount that device at a mountpoint. The file attachment is done with losetup:

    > sudo losetup --show -fL disk.img
    /dev/loop0

The -f option tells it to choose the next free loop device and --show tells it to report the chosen device. You can instead manually specify a loop device if you'd like:

    > sudo losetup -L /dev/loop0 disk.img

Provided the loop device in use is listed in your fstab, you can then mount it quite simply:

    > mount /dev/loop0

When you are done with the disk, unmount it and detach the loop device:

    > umount /dev/loop0 && sudo losetup -d /dev/loop0

If you use udisks2, you can use udisksctl to perform the same steps:

    > udisksctl loop-setup -f disk.img
    Mapped file disk.img as /dev/loop0.

    > udisksctl mount -b /dev/loop0
    Mounted /dev/loop0 at /mnt/loop0

The only real advantage to using udisks2 is if you are using a GUI wrapper like udiskie, wherein you can choose a loop device to attach from a menu. If it was attached as a superuser, it the wrapper will not be able to detach it. Using the 'loop-setup' command with udisks2 allows it to do the detaching as well. However, udisks2's underlying filesystem detection system seems to be a bit strict and the majority of internet-sourced PC98 disks that have been tested will not mount with udisks2, though they work fine with standard mount. Therefore, I recommend sticking with the native tools (losetup/mount).

## Mounting Floppy Disks
There are a plethora of floppy formats for the various PC98 emulators. The simplest formats are simply raw sector dumps or raw dumps with a header. As far as I can tell from my research, FDI is the only floppy format with a simple header. (D88 uses headers for each sector and FDD (Virtual98) seems to do something similar.) This limits which options we can work with directly, but there are tools which can convert these formats which can be run through Wine. It's not great, and a project for someday in the future would be to write Linux friendly converters or maybe even FUSE drivers...

But for raw and FDI formatted images, we can work with them easily enough. Raw images can be access using the examples above:

    > sudo losetup --show -fL disk.img
    /dev/loop0
    > mount /dev/loop0

An FDI image is similar, though we need to account for the 4k header by specifying an offset when attaching the device:

    > sudo losetup --show -fL -o 4096 disk.img
    /dev/loop0
    > mount /dev/loop0

And that's it!

## Mounting Hard Disks
It is possible to mount HD images, but it's a little more tricky, even for raw images. First of all, rather than mount the whole image, you mount individual partitions within that image. Granted, most setups will only have one partition, but it's something to keep in mind if your setup has multiple.

Basically, we need to identify the start of the partition within the image and specify that as the offset when mounting. The partition table begins at 0x200 within a raw image, and this site explains the table format beautifully:

https://sites.google.com/a/e-hdk.com/misc/pc-98-hdd

This is helpful, until you realize the partitions are addressed in the ancient Cylnder Head Sector system. (If you didn't have the privilege of specifying hard drive geometry in BIOS back in the day, you can read more about CHS here: https://en.wikipedia.org/wiki/Cylinder-head-sector ) This means that we can't know the actual start of the data unless we know the drive geometry. Those values were printed on the hard disk labels of the era and are usually specified in the header metadata of the various disk image formats, but when working with a raw image such information is not present.

In any case, determining the cylinders/heads/sectors and doing the math based on the partition table is just making things complicated. Instead, let's grep for the FAT filesystem header directly and use that! (Yes, that means the previous paragraphs were essentially useless, but surely a retro computer enthusiast such as yourself would find that information valuable and/or interesting.)

    IMAGE=imagefile.hdi; OFFSET=$(expr `grep -Ebaom 1 "FAT1[2|6]" ${IMAGE} | sed -E 's/:FAT1[2|6]//g'` - 54); sudo losetup --show -fL -o ${OFFSET} ${IMAGE}

This works for HDI, NHD and raw hard drive dumps. Note that it will only report the first partition and it wouldn't be reliable for finding later partitions, as the strings "FAT12" or "FAT16" are likely to be repeated within the same partition before reaching the next one. Your best bet in this case is to search for the FAT header yourself within the file and manually specify the offset.

You can also use a variation of this technique to directly mount the filesystem on your CF card hard disk "emulator" so you don't need to image the filesystem from/to the card each time you want to make simple changes:

    DEV=/dev/sdf4; OFFSET=$(expr `sudo grep -Ebaom 1 "FAT1[2|6]" ${DEV} | sed -E 's/:FAT1[2|6]//g'` - 54); sudo mount -o users,rw,exec,nofail,noauto,sync,check=strict,codepage=932,iocharset=euc-jp,x-gvfs-show,offset=${OFFSET} ${DEV} /mnt/pc98

Since the card is already mounted as a block device, there's no reason to do anything with the loop devices. You could add an fstab entry for this, but it's also possible the block device designation for the card reader could change, especially if you're using an external USB reader. It may be just as well to put the above into an alias/script and leave fstab alone. That choice is yours.

## Imaging From Disks
Making an image from a disk requires, of course, a floppy disk drive connected to your computer. USB based 3.5" disk drives are still commonplace, especially on auction sites, but older 5.25" floppy drives aren't really available on USB. Your best bet if you really need to work with 5.25s is to acquire an old drive and use a 34-pin FDD port to USB cable.

However you get the hardware setup, it should appear as a block device within your system. Imaging the disk is then just a matter of using dd:

    > sudo dd if=/dev/sdf of=new_image.img bs=128K status=progress oflag=sync

You can change the block size (the bs option) to something smaller for more frequent updates on the status, but the actual size doesn't matter.

Rather than working with physical floppy disks, though, you more than likely have CF based hard disk "emulator." These are the simplest to work with as all you need is any CF card reader, which you likely already have. To image your card, specify the device as the input and use dd again:

    > sudo dd if=/dev/sdf4 of=new_image.img bs=1M status=progress oflag=sync

Note that we're specifying the partition number here instead of the whole device like we did with a floppy disk. The reason for this is that, although it's marked as a "partition" within the block device, it actually represents the entire data of the PC98 disk. If instead we specify just the block device (/dev/sdf), it would capture the entire CF card. In my particular setup, I use 1GB CF cards though the PC98 hard drive size is only around 550MB or so. Specifying the device would capture that whole 1GB, wasting space and time.

You can check the block device/partition number associated with the card reader with `lsblk`.

    sdf           8:80   1 955.8M  0 disk
    └─sdf4        8:84   1   416M  0 part

Note that although the partition number is 4, only one partition is listed. I'm not familiar enough with the Linux kernel/filesystems to say why it does this, but it presents no problem when working with the data.

## Imaging To Disks
Writing an image to disk once again uses dd but also involves some concepts we previously discussed for mounting images. For flopp disks, if we're using an FDI, we need to specify the offset at which the raw data begins, i.e. the end of the header. Since the FDI header is 4K, we would do something like this:

    > sudo dd if=image.fdi of=/dev/sdf bs=128k status=progress oflag=sync skip=4096 iflag=skip_bytes

For a raw image, simply drop the skip and iflag options.

For a hard disk image, we don't need to fiddle with manually finding partitions, since we *want* the whole data (including the PC-98 format partition table) to appear on the device. However, as with the floppies, we do need to remove the image format header. HDI format also has a 4K header, so writing its image is exactly the same as the floppy:

    > sudo dd if=image.hdi of=/dev/sdf bs=128k status=progress oflag=sync skip=4096 iflag=skip_bytes

An NHD image has a 512 byte header:

    > sudo dd if=image.nhd of=/dev/sdf bs=128k status=progress oflag=sync skip=512 iflag=skip_bytes

And of course for a raw image, drop the skip and iflag options.

## Creating Floppy Images
Creating blank images is simple enough through an emulator, but we can also do it natively in Linux, just for fun.

Use dd, and set the count to the size of the disk in kb. For example, a 720k 3.5" disk:

    > dd if=/dev/zero of=disk.img iflag=fullblock bs=1K count=720 status=progress

a 1.2mb 5.25" disk:

    > dd if=/dev/zero of=disk.img iflag=fullblock bs=1K count=1200 status=progress

a 1.44mb 3.5" disk:

    > dd if=/dev/zero of=disk.img iflag=fullblock bs=1K count=1440 status=progress

(We've removed the oflag=sync option to speed things up, since we're working with the local filesystem instead of external devices. Run `sync` after dd if you're paranoid.)

If we attach the new file to a loop device, we can then format it to FAT:

    > sudo losetup --show -fL disk.img
    /dev/loop0
    > sudo mkfs -t msdos /dev/loop0

If working with a physical disk, you can do mkfs on the block device with the disk:

    > sudo mkfs -t msdos /dev/sdf

You'll stil need to mount the image in an emulator (or put the disk in your PC98) and run `sys` to set it up as a boot/system disk, however, so ultimately the create disk/format process is probably best done in an emulator (or on the PC98), but it's nice to know the options are there natively if need be.
