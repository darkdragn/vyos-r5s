## Initial image release

This is an initial draft. It currently uses the armbian uboot/kernel/firmware as the starting point.
The vyos arm squashfs was built primarily from Server-Server's repo and I'll add a link in for it in the future.
The system boot.cmd was modified to leverage vyos's custom live-boot hooks, and the initrd has them all imported.
The system will require a second partition with the label of persistence and a persistence.conf in it with the contents `/ union`

This is a quick snippet that will get you a working image:

```
########################
### Don't forget to fill this in ###
TARGET=/dev/sdX # Fill this in with your sdcard block device

cat vyos-r5s.img.gz | gunzip -c - |dd of=$TARGET bs=4M status=progress
printf "fix\n" | parted ---pretend-input-tty $TARGET print
parted --script $TARGET mkpart primary 3GB 100%
mkfs.ext4 -L persistence ${TARGET}2
mount ${TARGET}2 /mnt
echo "/ union" > /mnt/persistence.conf
umount /mnt
eject $TARGET
```
