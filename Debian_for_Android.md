# Debian & Android Together on G1 #
This does not replace Android. This also gives you access to the full plethora of programs available in Debian and let's you continue using your phone as it was intended to be: as an Android device with all the capabilities thereof.

Please note that this is not a "port": Debian already supports ARM EABI, which is the underlying architecture of Android.

**Update**: A status update on this work can be found on [this post](http://www.telesphoreo.org/pipermail/g1-hackers/2008-November/000032.html) from the G1-Hackers mailing list.

Ok, so Android exists. We finally have hardware, we even have source code. What we didn't have was some game changing moment whereby we suddenly got control of our mobile phones. You know, the moment that Google has promised us for the last year.

The reality of the situation is actually quite indicative of the attitudes in the cell phone world that even Google is powerless to affect. So, anyone who has actually gotten a G1 and has spent the time to try to do anything awesome with it have been utterly disappointed by the fact that we are just regular users on our own boxes: T-Mobile didn't give us root, being afraid of what we might do.

# Device Rooting/Jailbreaking #
To me, this particular limitation turns this device into nothing more than a toy: as anyone who has been following my adventures into iPhoneLand knows, I think it is a crying shame to be carrying around a high-speed ARM CPU running a modern OS with a reasonably large screen and numerous input methods to just be a sub-par cell phone.

Luckily, this all changed a couple days ago when someone found a serious flaw in the Android firmware: all keypresses are routed to the Linux console, which was running a root terminal. This meant that just typing ¶telnetd¶ into any program provides a very simple remote root shell.

Unfortunately, once there, it's actually a mite difficult to accomplish anything given Google's overly simplistic busybox replacement, toolbox. What we really need is a much more complete Unix userland. This device is powerful enough that we should be able to even develop directly on it.
# Installing Debian ARMEL #
The main thing I've so far seen on this matter have been a few attempts to get busybox on there. I, however, think we can go a lot further: following the instructions in this article will end you up with a full distribution of Debian, one of the most highly respected Linux distributions, and the ability to install almost anything you want.

To do this, we need to think through a few of the details of getting this sort of thing running on the G1. The first question: where do we put it? The device has some internal flash, but it isn't really enough: only 128MB to share with the OS and other applications.

We therefore turn our attention to the much more reasonably sized microSD card, a format which lets us get up to 16GB of space. Unfortunately, for compatibility with all existing readers, these cards are formatted FAT, which makes them nearly useless to store Unix programs and data on.

This is where we have to start getting tricky: we could put our Debian root inside of a filesystem image that we, in turn, store on the SD card as a single file on the FAT partition. To do this, we just need to mount the file over a loopback driver with a more reasonable filesystem.

Checking /proc/filesystems, we find all kinds of filesystems we could choose from: vfat, yaffs, yaffs2... ok, or not. T-Mobile only installed the small handful of filesystem drivers that Google needs to make Android function. This means we need to load our own driver, which is finally where we get our break: the kernel is setup for modules.

If there is any demand, I will write up a howto on compiling kernel modules for the G1. I encourage people who want to discuss this kind of G1 development/hacking to join [the G1-Hackers mailing list](http://www.telesphoreo.org/cgi-bin/mailman/listinfo/g1-hackers) which I'm hosting at Telesphoreo.

# Building the Debian Image #
Thankfully, Debian already [fully supports ARM EABI](http://wiki.debian.org/ArmEabiPort) and even has [a helpful guide](http://wiki.debian.org/ArmEabiHowto) on doing this installation.

For people who would prefer to just use a ready-made image, I constructed this filesystem image for a 750MB root and have uploaded the [final image file](http://g1tools.googlecode.com/files/debian-armel-750.img.bz2). This file has been bzip2 compressed down to about 85MB.

As RapidShare is rather slow, [modmyGphone.com modmyGphone.com] has graciously mirrored the image on their servers at a few different URLs. Details to be found at [their summary of this article.](http://modmygphone.com/forums/showthread.php?t=5191)

If you would rather make this image yourself (maybe you would like a different size than 750MB, or just want more control over the process), here is the set of commands used to construct it, consolidated into one place. Note the 749999999, dd is irritating.

apt-get install debootstrap dd if=/dev/zero of=debian.img seek=749999999 bs=1 count=1 mke2fs -F debian.img mkdir debian mount -o loop debian.img debian debootstrap --verbose --arch armel --foreign lenny debian [http://ftp.de.debian.org/debian](http://ftp.de.debian.org/debian) umount debian

# Building our Debian "Kit" #
Please note that the initial version of these instructions didn't take into account that I was still using a relatively rare firmware version: RC19. I have added variants of these modules for different firmwares.

If you are using a version newer than RC30, then you should be able to find the required kernel modules already on your filesystem: they come with JF and Haykuro's firmware images.

Once we have our image, we need to transfer it to our microSD card. At the same time we should also grab a few other files we need. One important aspect of this is that these steps require different kernel modules depending on what version of the firmware you are using (as the configuration slightly changes over time).

  * ext2.ko　　([RC19](http://cache.saurik.com/android/2.6.25-01828-g18ac882/fs/ext2/ext2.ko)) ([RC29/30](http://cache.saurik.com/android/2.6.25-01843-gfea26b0/fs/ext2/ext2.ko)) - the standard Linux filesystem driver
  * unionfs.ko  ([RC19](http://cache.saurik.com/android/2.6.25-01828-g18ac882/fs/unionfs/unionfs.ko)) ([RC29/30](http://cache.saurik.com/android/2.6.25-01843-gfea26b0/fs/unionfs/unionfs.ko)) - lets us merge folders together (advanced)
  * [busybox](http://cache.saurik.com/android/armel/busybox) - for a few key tools we need working variants of

Put all of these (and debian.img) together in a folder on the microSD card (I do this using the USB connection, which I find simple and fast to use). Note that if you downloaded a premade image you might want to rename it to debian.img to make these instructions simpler/work.

# Setting up the Mount #
Now for some more decisions: where do we, and where did we, put things? I placed the kit in a folder under the root directory of the microSD card and put everything else in /data/local (the one useful folder you can normally write to).

To make the remaining instructions simpler, I am going to export these paths as environment variables so they are easy to use later without having to retype them. It also makes the remaining instructions semantically much more simple to read.

export kit=/sdcard/kit export bin=/data/local/bin export mnt=/data/local/mnt

Next, there are a few environment variables we really need to setup lest we, or some of the software we try to use, goes insane. We will also take this time to quickly load the ext2 filesystem driver so we have it available in later steps. Note that these might look weird (HOME in particular), but ignoring these environment variables can cause major problems later.

export PATH=$bin:/usr/bin:/usr/sbin:/bin:$PATH export TERM=linux export HOME=/root insmod $kit/ext2.ko

Our next order of business is to copy busybox to somewhere it can be executed. As the G1 doesn't come with a cp command, we have to use cat to do this. Note that the mkdir command I have there cannot have a -p (as toolbox doesn't support it), so if you chose a crazy deep folder (or one that exists) you may have to do ignore some errors or do something more intelligent.

mkdir $bin #-p cat $kit/busybox >$bin/busybox chmod 755 $bin/busybox

Now that we have busybox, we can use it to create a device node for a loopback driver. We need busybox for this as the G1 does not come with mknod, which is needed to do this. We will also alias busybox to something shorter, making it easier to type. (Often you would have busybox construct a set of symlinks for its subcommands, but I think that's overkill for our temporary usage.)

alias _=busybox_ mknod /dev/loop0 b 7 0

If you get an error while running _that it isn't found, try just resetting it._

unalias _alias_=busybox

Finally, we get to mount the image! Note to use the filename you used for it. We will mount this "noatime" in order to minimize unneccessary writing to the flash memory part (which is both slow and will decrease its lifetime). Thanks goes to Lauren Weinstein for reminding me of that flag!

_mkdir -p $mnt_ mount -o loop,noatime $kit/debian.img $mnt

There are a few common error cases here: if you get a complaint about /etc/fstab then $kit/debian.img probably does not exist, and if you get a usage dump then $mnt is likely unset.

# Finalizing the Installation #
At this point, we have to go back and finish some of the work that debootstrap isn't able to handle. First, we have to run some package scripts and fix the URL of the main Debian repository (debootstrap messes this up for some unknown reason). If you downloaded the prebuilt image, these two commands don't need to run.

_chroot $mnt /debootstrap/debootstrap --second-stage echo 'deb http://ftp.de.debian.org/debian lenny main' >$mnt/etc/apt/sources.list_

To make certain that networking works fully (specifically DNS resolution) we have to verify the nameserver listed in /etc/resolv.conf. One easy way to do this for a HOWTO is to just set this DNS server to a known working value. I have already done this in the pre-made image file. You might want to do something more intelligent here.

echo 'nameserver 4.2.2.2' >$mnt/etc/resolv.conf

Now that that's over (which probably took almost ten minutes), we can enter the Debian environment using chroot. This will give us a shell that is locked into Debian land, and capable of running all the advanced/awesome things we can do there. You should continue from here even if you have a premade image.

_chroot $mnt /bin/bash_

Once in, we need to do a quick few mounts to make things fully functional.

mount -t devpts devpts /dev/pts mount -t proc proc /proc mount -t sysfs sysfs /sys

Next we have a simple change that I should have done in the preconstructed image, but did not realize I needed to do, and now I've already distributed it, so maybe next time I make an image it will already be done.

rm -f /etc/mtab ln -s /proc/mounts /etc/mtab
We should also set a password for the root account so we can remotely log into it later.

passwd root

# Setting up OpenSSH #
Here is where the true power of Debian comes through: getting more advanced software rapidly onto the phone. Let's start by getting SSH up so we can get a fully functional terminal (it might be hard to notice, but there are definitly a few issues running through the telnetd into the chroot).

To start, we use APT (Debian's package tool) to first update its catalog and then install the server. The second command will also automatically start the server.

apt-get update apt-get install openssh-server

If you come back later and would like to start OpenSSH again (as we aren't doing the normal Debian bootup process), you can use the following command (optionally using "restart" instead of "start").

/etc/init.d/ssh start

# Ok, Now What? #
Now you have fun! The first things I got on there were subversion, gcc, and vim so I could start developing for it. Again, people who want to get deep into the internals of Android (kernel drivers, hardware access, flashing) should join [the G1-Hackers mailing list](http://www.telesphoreo.org/cgi-bin/mailman/listinfo/g1-hackers).
If you later reboot your device, you will need to run some, but not all, of these commands again. A careful read-through will make this clear, but if you happened to use the same paths that I did in my example you can use [this script](http://cache.saurik.com/android/script/remount.sh) to re-setup the mounts.

It should be noted that, while all of this is operational, Android will be unable to mount over the USB cable as we are using the microSD card to run Debian. To fix this, you have to shut down the mounts we setup.

umount $mnt/dev/pts $mnt/proc $mnt/sys $mnt _losetup -d /dev/loop0_

Note the losetup! Skipping this step will leave the loopback device we created allocated, which will in turn continue to block Android's ability to reuse the microSD card for mounting over USB.

Running Debian Code at /

Ok, so one thing that was unfortunate about all of this is that we pretty much have to make a choice: in Debian, or in Android. This is the kind of choice we really shouldn't have to make, especially given that the two systems pretty much don't overlap. This is where unionfs comes in.

Note that all commands from here on are being executed outside of the Debian chroot we entered earlier, so still over telnet. You can exit with exit or just open a new telnet session. You can then download these commands as a [ready-made script](http://cache.saurik.com/android/script/unionfs.sh).

To run this ready-made script you should already have exported your standard environment variables. If you have the script in your $kit, then you can run it as so with . (so you can avoid issues regarding noexec on /sdcard).

. $kit/unionfs.sh

As you are likely already deep into these instructions by now and no longer wish to disconnect everything to install that file, note that you can just wget it with your Debian environment.

_chroot $mnt wget -O /tmp/unionfs.sh http://cache.saurik.com/android/script/unionfs.sh cat $mnt/tmp/unionfs.sh >$kit/unionfs.sh rm $mnt/tmp/unionfs.sh
insmod $kit/unionfs.ko mount -t unionfs -o dirs=$mnt/etc=rw:/etc=ro unionfs /etc_

What this does is make /etc contain both the files from Android and the files from Debian. It also sets the system up so that if we modify any files in /etc (or create any new ones) these modifications will get stored in our Debian partition: a feature that now gives us a fully working /etc!

The next problem is that Android and Linux use different naming conventions for their dynamic linker. On Android we have /system/bin/linker, whereas on Linux we have /lib/ld-linux.so.3. This means we get file not found errors just from running valid software. This is easy enough to fix with a symlink.

_mount -o remount,rw /_ ln -s $mnt/lib/

At first glance this might seem dangerous, but it isn't. While we are modifying the root filesystem of the device (something we aren't supposed to be able to do), it happens to be "rootfs": a special instance of the Linux ramfs filesystem. This means that any changes we make to it are undone by a simple reboot.

At this point we should be able to run most Debian programs without entering a chroot by just running the program from $mnt. Unfortunately, not everything is going to work as most of the files are in the "wrong" place. Let's fix that with some more symlinks.

for x in \ bin boot home media mnt \ opt selinux srv usr var do _ln -s $mnt/$x / done_

This leaves only a few folders that we need to deal with. The first one is trivial: /root is empty, so we can just get rid of it and replace it with another symlink. Also, as we are now done modifying the filesystem on /, I highly recommend reprotecting this mount as files that end up here directly take up RAM (not flash) and do not get sync'd back to the Debian environment (which is confusing/wrong).

rmdir /root _ln -s $mnt/root /_ mount -o remount,ro /

This leaves /sbin and /dev. The former can be handled by a simple unionfs, but the situation with /dev is actually pretty evil. It has a mount underneath it, /dev/pts, that you seemingly can't layer under a unionfs for whatever reason and expect it to still work. The fix for this is to remount it back on top after the union.

mount -t unionfs -o dirs=$mnt/sbin=rw:/sbin=ro unionfs /sbin mount -t unionfs -o dirs=$mnt/dev=rw:/dev=rw unionfs /dev mount -t devpts devpts /dev/pts

At this point we have everything setup well enough that even things like OpenSSH should work, so let's restart it in this environment.

/etc/init.d/ssh restart

# Cavaets and Open Issues #
While this all looks great, there are a few issues worth mentioning:

The first is that I am not certain where usernames and other authentication details are coming from, but wherever it is Debian is going to need to be taught how to get access to that same information through PAM or some other architecture. If you use Debian's ls to look at files you will note that it doesn't know anyone's username.

Secondly, using a symlink tree for / rather than pivoting it to the mounted image (which I couldn't come up with a clean way to do given that the system has already booted and I'm no longer init) means that any package that tries to add a folder to / is going to fail to install: you might want to use apt-get mostly from within the chroot.

Finally, this is seriously going to be very temporary. Updates for RC30 are already going out which fix the console bug that let us do all this in the first place (as you need root access to get files into places where you can run them). I therefore highly recommend playing with it while you still can.

(Frowny pants on that, by the way :(. I wish I understood why cell phone companies are so keen on selling us devices that are purposely dumbed down. Is it actually good for their business?)