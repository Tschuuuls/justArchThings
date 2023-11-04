# Sane low memory config on Arch using zram
PHILIPS LAPTOP HAT 62%


## 0. Check the initial swap setup
1. Run swapon --show` and save make note of the output
## 1. Disable ZSWAP (should be on by default)
ZSWAP is a alternate method to using ZRAM. Using both at the same time is not advised
1. Check if it's even on: `cat /sys/module/zswap/parameters/enabled` should return 1
2. Disabling ZSWAP permanently by setting a kernel parameter (if you don't use GRUB, look it up): 
    - `vi /etc/default/grub`
    - Edit the `GRUB_CMDLINE_LINUX_DEFAULT="..."` line and add `zswap.enabled=0` `mem_sleep_default=deep` at the end

## 2. Enable ZRAM
1. `vi /etc/modules-load.d/zram.conf` and add zram in the file to load the zram kernel module at boot
2. Set up udev rules to configure ZRAM by `vi /etc/udev/rules.d/99-zram.rules` and add the following to the file:
`ACTION=="add", KERNEL=="zram0", ATTR{comp_algorithm}="zstd", ATTR{disksize}="4G", RUN="/usr/bin/mkswap -U clear /dev/%k", TAG+="systemd"'`
you could change the compression algo by changing ATTR{comp_algorithm}= and you should change the ATTR{disksize}= which is your ZRAM size. ZRAM size should be, according to [Pop!\_OS](https://github.com/pop-os/default-settings/pull/163): half the size of your ram in GB. Fedora uses a max of 8G.

3. Add your new ZRAM volume to your fstab aka the volumes you mount at boot time: `vi /etc/fstab` and add `/dev/zram0 none swap defaults,pri=100 0 0` to the end. Note: you can't refer to a ZRAM volume by it's UUID!

## 3. Set up ZRAM pritorities
We need to set the priorities on which the system decides to use the new zram swap
1. Open `vi /etc/sysctl.d/99-vm-zram-parameters.conf` and add: 
``` c++
vm.swappiness = 180
vm.watermark_boost_factor = 0
vm.watermark_scale_factor = 125
vm.page-cluster = 0 
```
These are the values from Pop!_OS, they should work.
Some more info from the ArchWiki
>"The default value [for vm.swappiness] is 60. For in-memory swap, like zram or zswap, as well as hybrid setups that have swap on faster devices than the filesystem, values beyond 100 can be considered. For example, if the random IO against the swap device is on average 2x faster than IO from the filesystem, swappiness should be 133 (x + 2x = 200, 2x = 133.33)."
>
>On a system with a hard drive, random IO against the zswap device would be orders of magnitude faster than IO against the filesystem, so swappiness should be ~200. Even on a system with a fast SSD, a high swappiness value may be ideal. 

2. Reboot
3. This should give you working ZRAM support, but hibernate is probably broken now :crying_cat_face: `zramctl` should tell you if ZRAM works

## 4. Fixing your hibernation
Since you can't hibernate to your RAM (because it's powered off when in sleep mode and looses all it's data) you need to supply a "normal" swap partition or file to hibernate to.

I assume you had a swap FILE and not a partition, don't do this if you had a partition set up.

1. Install https://github.com/gissf1/zram-hibernate, the aur package should be https://aur.archlinux.org/packages/zram-hibernate-git by running `yay -S zram-hibernate-git'
2. Create the file `/etc/zram-hibernate.conf` and add `KERNEL_SWAP_DEVICE=/swapfile.swp` the swapfile here should be your original swap file from **0.1** If you don't have one create one like this: https://wiki.archlinux.org/title/Swap#Swap_file
3. Check it by running `systemctl hibernate`. Wake the computer after a bit and check the log `sudo journalctl -n 60` to see if any errors pop up.
