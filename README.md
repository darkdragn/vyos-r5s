## Initial image release

This is an initial draft. It currently uses the armbian uboot/kernel/firmware as the starting point.
The vyos arm squashfs was built primarily from Server-Server's repo and I'll add a link in for it in the future.
The system boot.cmd was modified to leverage vyos's custom live-boot hooks, and the initrd has them all imported.

In the v2 I've added a 1Gb persistence partition to make testing easier on me. You can expand it to fill your sdcard if you want, but it should mostly be for logs and config files, so 1Gb is enough for me to see if things are working.

This is a quick snippet that will get you a working image:

```
########################
### Don't forget to fill this in ###
TARGET=/dev/sdX # Fill this in with your sdcard block device

cat vyos-r5s-v2.img.gz | gunzip -c - |dd of=$TARGET bs=4M status=progress
sync
eject $TARGET
```
