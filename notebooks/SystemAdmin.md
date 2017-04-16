---
layout: default
title: Some notes on computer system administration
---

# Linux/Ubuntu OS

## Diagnose display problems and install the default NVidia driver
The video card driver and display settings are usually tricky on Linux.
I have encountered black-screen and dual-monitor display problems while installing and reinstalling Ubuntu 16.04 GNOME and Unity systems on my Lenovo P50 mobile workstation and other computers.
Fortunately, I have received helps on the [ubuntu forum](https://ubuntuforums.org/showthread.php?t=2323113) and other places from experts and fixed those past issues.

If a black screen happened in the startup after installation, and in the TTY mode (Ctl+Alt+F1) it shows "a start job is for hold until boot finishes up..." but it never finishes, it might be related to the display settings and drivers.
The following worked for me.
Boot into the recovery mode with *Network* or *run commands as root* from the ***Advanced Ubuntu Options***, and run
```
sudo apt-get remove plymouth
sudo apt-get remove xserver-xorg-video-intel
```
and restart with `sudo reboot`. Certainly, no need to use `sudo` when running as root.
If it works, then reinstall `xserver-xorg-video-intel`.
If problem persists, try `sudo purge nvidia*` and reboot and reinstall the default NVidia driver by `sudo ubuntu-drivers autoinstall`.
Pay attention to the build message to see if there is any error in the process of building the driver into the Linux kernel modules.
Then reconfigure the `lightdm` by
```
sudo apt-get --reinstall install gdm
sudo dpkg-reconfigure lightdm
sudo service lightdm start
```
It should now start a graphic window, or reboot to see the effect.

For external monitor problems, it could be related to the `xserver-xorg` setting and NVidia drivers.
To diagnose the issue, here are some useful commands:
`cat .xsession-errors` in the user's home directory to see if there is any error like `openConnection: connect: No such file or directory`. That means `ls /etc/X11/` will not show a file named as `xorg.conf`.
This usually implies the NVidia driver and the xorg files are not correctly built.
Run `dpkg -l | grep linux-` to see all the kernel installed on the system, and then `dpkg -l | grep -i nvidia` and `dkms status` to see what version of NVidia driver has been installed.
If `lsmod | grep nvidia` gives nothing, if means the NVidia driver is not built into the current kernel (use `uname -a` to find the current kernel series number).
So, the solution is to rebuild the NVidia driver using default settings by
```
sudo apt purge nvidia*
sudo rm /etc/X11/xorg.conf # Run only when xorg.conf is there.
sudo ubuntu-drivers autoinstall
```
The building process will be configuring all installed Linux kernels.
Try to see if there is any error message for building it for each kernels.
If a message shows the current kernel doesn't support the NVidia driver, a working kernel has to be used to boot up the computer and the non-compatible kernel shouldn't be used for the compatibility reason.
After a successful rebuilding process, in the terminal run `sudo service lightdm restart`, and it should bring in the graphic window or try to reboot to see the effect.

For other issues, `dpkg -l nvidia-prime` and `cat /var/log/Xorg.0.log` should show some basic message to help identify where the problem is related to the video drivers.
To confirm problem solved at the root, run
```
lspci -k | grep -EA2 'VGA|3D'
sudo lshw -C display
```
you should find both Intel integrated driver and the concrete NVidia display drivers are installed to the corresponding modules.
The `nvidia-PRIME` package should also be able to run and configure how the GPUs are used on the computer.

*PS*: after Ubuntu 16.10, GNOME has developed better display management systems (like [gnome-wayland-session](http://www.phoronix.com/scan.php?page=news_item&px=Ubuntu-GNOME-16.10-Wayland)) and the configuration and debugging process might be different from the above.

## Ubuntu 16.04 hangs to shut down
This problem occurs when I freshly installed the Ubuntu 16.04 Gnome system.
Whenever I press the Shutdown button on the Power Off menu, it will take up to 1min 30sec to respond.

Turns out, it is related to the CUPS remote printers from the cups-browsed service which automatically add network printers to the computer.
I don't need this automatic printer adding function and hence disabled this service, and then everything works fine again.
Reference is [here](http://askubuntu.com/questions/760952/slow-shutdown-on-ubuntu-16-04-lts-stopping-thermal-daemon-running-fit-make-remo).

In most cases, a command will save the time regardless of the hanging-out issue.
`sudo reboot` is enough to reboot the computer as soon as possible.
To shutdown (not halt), [here](https://askubuntu.com/questions/578144/why-doesnt-running-sudo-shutdown-now-shut-down) are the commands available:
```
sudo shutdown -h now
sudo shutdown -P now
sudo poweroff
sudo halt -p
sudo init 0
```
The `poweroff` and `halt` commands basically invoke `shutdown` (except for the `poweroff -f`). `sudo poweroff` and `sudo halt -p` are exactly like `sudo shutdown -P now`. The command `sudo init 0` will take you to the runlevel 0 (shutdown).

Now what if we want to shut down forcefully, i.e., we don't want to wait for processes to close normally? In that case you can use:
```
sudo poweroff -f
```
This will not use shutdown. Rather, it will invoke the [reboot(2)](http://manpages.ubuntu.com/manpages/zesty/en/man2/reboot.2.html) system call (used for reboot, poweroff & halt) to power off the computer instantly.


*Update*: Without disabling the `CUPS` service, this bug seems having been fixed with `cups-filters` v1.11.4-1 yet not released in the official Ubuntu 16.04 repository. A workaround solution to install the latest version of `cups-filters` and its dependencies can be found in [this solution](http://askubuntu.com/a/896655/390708).

## Automatically mount partitions at startup
The graphic approach is suggested [here](https://askubuntu.com/questions/598036/automatic-ntfs-partition-mount-on-startup)--that is the follow.
Into ***Disks*** software, select the partition that you want to mount at startup.
Click its `Setting`, select `Edit Mount Options`, and then select `Mount at Startup` and fill in with corresponding mounting information.

If preferring to modify the configuration file, `/etc/fstab` is the place to play with.
Filling in the mounting partition information following the instructions [here](https://help.ubuntu.com/community/Fstab), [here](https://help.ubuntu.com/community/AutomaticallyMountPartitions) or [here](https://help.ubuntu.com/community/MountingWindowsPartitions).
For example, one line can be
```
UUID=BC34947B34943A7A /media/D ntfs-3g defaults,x-gvfs-name=D 0 0
```
The `UUID` information can be found by running `sudo blkid` on a terminal.

## Unable to mount NTFS disks automatically on startup.
When the Ubuntu 16.04 started, I got the following error message:
```
Unable to access “My Drive”

Error mounting /dev/sdb4 at /media/D Center: Command-line `mount -t "ntfs" -o "uhelper=udisks2,nodev,nosuid,uid=1000,gid=1000,dmask=0077,fmask=0177" "/dev/sdb4" "/media/D Center"' exited with non-zero exit status 14: The disk contains an unclean file system (0, 0).
Metadata kept in Windows cache, refused to mount.
Failed to mount '/dev/sdb4': Operation not permitted
The NTFS partition is in an unsafe state. Please resume and shutdown
Windows fully (no hibernation or fast restarting), or mount the volume
read-only with the 'ro' mount option
```
A solution according to [this source](http://askubuntu.com/questions/462381/cant-mount-ntfs-drive-the-disk-contains-an-unclean-file-system) is to run the following command, for example,
```
sudo ntfsfix /dev/sdb4
```

The root cause of this issue might be related to the fast-booting feature of Windows 10.
Therefore, to fix it for all future events, the fast-booting feature should be turned off as instructed [here](http://superuser.com/questions/1152001/shutdown-windows-10-truly-for-a-dual-booting-system).
A detailed explanation on this issue can be found [here](http://askubuntu.com/questions/145902/unable-to-mount-windows-ntfs-filesystem-due-to-hibernation).

## Keep Linux kernel updated on a LTS Ubuntu OS
Normally, a LTS Ubuntu OS will stick to a particular kernel series to keep the OS stable.
However, this doesn't mean the Ubuntu OS doesn't update their supported Linux kernel series on their repository.
In fact, for example, Ubuntu-16.04-1 was released with kernel 4.4.X, but then Ubuntu-16.04-2 was released with kernel 4.8.X.
The default behavior of a local Ubuntu-16.04-0 distribution is to stick to the initially installed kernel series before the OS is upgraded to a new distribution like Ubuntu 18.04 LTS. To change this default behavior and keep receiving updated Ubuntu's officially supported kernel series on a LTS Ubuntu distribution, one can enable the HWE stacks by installing the following packages:
```
sudo apt install linux-generic-hwe-16.04 linux-lowlatency-hwe-16.04
```
according to the [official rolling LTS enablement stack page for Ubuntu 16.04 LTS distribution](https://wiki.ubuntu.com/Kernel/RollingLTSEnablementStack).
The kernel update will keep rolling in the first two years after the distribution was initally released.

## Remove old Linux kernels to save space in /boot
One common problem of upgrading linux kernel is "no enough space" in the /boot partition. This might be due to the old kernels and the new kernel files copied to /boot while compiling.
One line of command to remove those unnecessary files is
```
dpkg -l linux-{image,headers}-"[0-9]*" | awk '/^ii/{ print $2}' | grep -v -e `uname -r | cut -f1,2 -d"-"` | grep -e '[0-9]' | xargs sudo apt-get -y purge
```
Reference: [the solution on Askubuntu](http://askubuntu.com/questions/89710/how-do-i-free-up-more-space-in-boot).

## Mouse scrolls too fast on Chrome browser
If one scroll of the mouse on the Chrome brower can go a far distance on a page, it might have been affected by [this Ubuntu bug](https://bugs.launchpad.net/ubuntu/+bug/971321).
This only affects Chrome and a few wireless mouse brands including Microsoft.
A workaround before the bug is fixed is to unplug and replug the wireless receiver of the mouse.

## Using LVM and btrfs filesystem to partition disks
LVM is a good partition and logical volume management tool to organize disk space, especially for the cases that one Linux directory is spanned onto two physical disks or is to be extended to extra disks in the future.
On the other hand, `btrfs` filesystem is very flexible and convenient to make snapshot backups and secure & safe to manage system files in Linux.
Combining these two, we can have a very flexible and convenient way to run Linux.
Below are some common scenarios that I have encountered to use LVM with the btrfs filesystem.

### I. creating LVM managed volume groups and logical volumes for installing Ubuntu along side with Windows.
To set up the partitions, I freed out ~130GB space in my case under Windows 10 using disk managing tools.
As another preparation step, I went into BIOS, and disabled Secure Boot (Startup) and enbled CSM on my Thinkpad P50 laptop.
Then booted in an Ubuntu USB liveCD, I used Gparted to set up SWAP (4GB), /boot for 350MB, and the rest on SSD (`/dev/sdb`) as LVM2 file management system with btrfs filesystem formate. Then in the command line, I created an LVM Physics Volume from GParted btrfs partition space `/dev/sdb10` and two logical volumes for / and /home under the volume group `ubuntuvg` which only includes `/dev/sdb10`:
```
sudo pvcreate /dev/sdb10
sudo vgcreate ubuntuvg /dev/sdb10
sudo lvcreate -n root -L 25g ubuntuvg
sudo lvcreate -n home -l 100%FREE ubuntuvg
```
That is 25GB btrfs for / and 120GB btrfs for /home, both of which are extendable in the future event of upgrading disks.
They are addressed at `/dev/mapper/ubuntuvg-root` and `/dev/mapper/ubuntuvg-home` on the disk filesystem record (run `df -h` to see them once mounted).
On the system partition table as recognized by `mount`, for example, they are recognized as `/dev/ubuntuvg/root` and `/dev/ubuntuvg/home`, respectively.

Then I installed the Ubuntu OS on those partitions, and installed the bootloader to the UEFI partition (`/dev/sdb7` where the Windows 10 bootloader was installed before) in my case.

To permanently add those volumes to be automatically mounted at startup of the Linux system, one can edit the `/etc/fstab` file to include them.
My file has the following partial content:
```
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
/dev/mapper/ubuntuvg-root /               btrfs   defaults,subvol=@ 0       1
# /boot was on /dev/sdb9 during installation
UUID=5cd44ab2-7e1f-4886-9d4e-c1a6e10348ad /boot           ext4    defaults        0       2
# /boot/efi was on /dev/sdb7 during installation
UUID=6226-94B9  /boot/efi       vfat    umask=0077      0       1
/dev/mapper/ubuntuvg-home /home           btrfs   defaults,subvol=@home 0       2
/dev/sdb8       none            swap    sw              0       0
/dev/disk/by-uuid/BC34947B34943A7A /media/D auto nosuid,nodev,nofail,x-gvfs-show,x-gvfs-name=D 0 0
```
where the UUIDs can be obtained by running `sudo blkid`.

### II. Reallocate space between logical volumes
To resize / and /home as logical volumes defined above, I ran the following commands to reallocate a 10G space from /home to / while booted into the USB Ubuntu liveCD system.
The idea is to reduce the physical directory space using btrfs filesystem tools before to use `lvresize` command to reduce the logical volume space for /home, and then to increase the logical volume space for / before to increase the btrfs filesystem physical space next.
Using btrfs resizing tool needs to have the physical partitions mounted to a system directory, which is different from ext4 and other filesystems; while resizing logical volumes on LVM usually requires those logical volumes unmounted from the system to be safe.

```
sudo vgdisplay -v # to preview the physical volumes, virtual groups and logical volumes from LVM.
sudo mount -t btrfs /dev/ubuntuvg/home /mnt
sudo btrfs filesystem resize -10G /mnt
sudo umount /mnt
sudo lvresize -L -10G ubuntuvg/home
sudo lvresize -L +10G ubuntuvg/root
cd /dev
sudo mkdir media
cd media
sudo mkdir root
sudo mount -t btrfs /dev/ubuntuvg/root /dev/media/root
sudo btrfs filesystem resize +10G /dev/media/root
sudo mount /dev/ubuntuvg/home /mnt
df -h
sudo vgdisplay -v
```

The last step had verified that all of the commands worked out for this resizing operation between / and /home logical volumes.

# Notes on using some common tools
## Git
* One of the common scenario of using Git is to make fixes to a repo *A* not belonging to me but I have already customized the source in the repo for my personal usage before hand.
In this case, I can make a new branch called *patch-1*, for example, in my personal repo just for the fix:
```
git remote add A <url of A>
git fetch A
git checkout -b patch-1 A/patch-1
```
Once I commit all necessary fixes, I can make a pull request through the web interface of the Git repository.
After the PR is merged, I can then delete the *patch-1* branch remotely and locally by
```
git push origin --delete patch-1
git branch -d patch-1
```