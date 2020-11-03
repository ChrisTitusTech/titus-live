# Titus-Live - A Live Environment Built for Recovering and Building PCs

## IN DEVELOPMENT - DO NOT USE

## Pre-Requisites

Have a functional Debian-Based System with these packages installed

```
sudo apt install \
    debootstrap \
    squashfs-tools \
    xorriso \
    isolinux \
    syslinux-efi \
    grub-pc-bin \
    grub-efi-amd64-bin \
    git \
    mtools -y
```

Clone This repository into your Home directory
```
cd ~
git clone https://github.com/ChrisTitusTech/titus-live
```

## Building the CHROOT Environment

```
sudo debootstrap \
    --arch=amd64 \
    --variant=minbase \
    bullseye \
    $HOME/titus-live/chroot \
    http://ftp.us.debian.org/debian/
```

Jump into the CHROOT of our Image `sudo chroot $HOME/titus-live/chroot`

## Editing The Image in CHROOT

All Commands below are entered via CHROOT

```
echo "titus-live" > /etc/hostname
apt update && \
apt install --no-install-recommends \
    linux-image-amd64 \
    live-boot \
    systemd-sysv \
    network-manager net-tools wireless-tools wpagui \
    curl openssh-client \
    blackbox xserver-xorg-core xserver-xorg xinit xterm \
    nano vim -y
```

Set Root Password `passwd root`

exit

## Packaging the Image

```
mkdir -p $HOME/titus-live/{staging/{EFI/boot,boot/grub/x86_64-efi,isolinux,live},tmp}
sudo mksquashfs \
    $HOME/titus-live/chroot \
    $HOME/titus-live/staging/live/filesystem.squashfs \
    -e boot
cp $HOME/titus-live/chroot/boot/vmlinuz-* \
    $HOME/titus-live/staging/live/vmlinuz && \
cp $HOME/titus-live/chroot/boot/initrd.img-* \
    $HOME/titus-live/staging/live/initrd
```

### Make MBR Boot Menu

```
cat <<'EOF' >$HOME/titus-live/staging/isolinux/isolinux.cfg
UI vesamenu.c32

MENU TITLE Boot Menu
DEFAULT linux
TIMEOUT 600
MENU RESOLUTION 640 480
MENU COLOR border       30;44   #40ffffff #a0000000 std
MENU COLOR title        1;36;44 #9033ccff #a0000000 std
MENU COLOR sel          7;37;40 #e0ffffff #20ffffff all
MENU COLOR unsel        37;44   #50ffffff #a0000000 std
MENU COLOR help         37;40   #c0ffffff #a0000000 std
MENU COLOR timeout_msg  37;40   #80ffffff #00000000 std
MENU COLOR timeout      1;37;40 #c0ffffff #00000000 std
MENU COLOR msg07        37;40   #90ffffff #a0000000 std
MENU COLOR tabmsg       31;40   #30ffffff #00000000 std

LABEL linux
  MENU LABEL Titus Live [BIOS/ISOLINUX]
  MENU DEFAULT
  KERNEL /live/vmlinuz
  APPEND initrd=/live/initrd boot=live

LABEL linux
  MENU LABEL Titus Live [BIOS/ISOLINUX] (nomodeset)
  MENU DEFAULT
  KERNEL /live/vmlinuz
  APPEND initrd=/live/initrd boot=live nomodeset
EOF
```

### Make UEFI Boot Menu

```
cat <<'EOF' >$HOME/titus-live/staging/boot/grub/grub.cfg
search --set=root --file /DEBIAN_CUSTOM

set default="0"
set timeout=30

# If X has issues finding screens, experiment with/without nomodeset.

menuentry "Titus Live [EFI/GRUB]" {
    linux ($root)/live/vmlinuz boot=live
    initrd ($root)/live/initrd
}

menuentry "Titus Live [EFI/GRUB] (nomodeset)" {
    linux ($root)/live/vmlinuz boot=live nomodeset
    initrd ($root)/live/initrd
}
EOF
```

### Grub Menu Loader

```
cat <<'EOF' >$HOME/titus-live/tmp/grub-standalone.cfg
search --set=root --file /DEBIAN_CUSTOM
set prefix=($root)/boot/grub/
configfile /boot/grub/grub.cfg
EOF
```

```
touch $HOME/titus-live/staging/DEBIAN_CUSTOM
cp /usr/lib/ISOLINUX/isolinux.bin "${HOME}/titus-live/staging/isolinux/" && \
cp /usr/lib/syslinux/modules/bios/* "${HOME}/titus-live/staging/isolinux/"
cp -r /usr/lib/grub/x86_64-efi/* "${HOME}/titus-live/staging/boot/grub/x86_64-efi/"
```

### Build Grub 

```
grub-mkstandalone \
    --format=x86_64-efi \
    --output=$HOME/titus-live/tmp/bootx64.efi \
    --locales="" \
    --fonts="" \
    "boot/grub/grub.cfg=$HOME/titus-live/tmp/grub-standalone.cfg"
```

### Boot IMG File

```
cd $HOME/titus-live/staging/EFI/boot && \
    dd if=/dev/zero of=efiboot.img bs=1M count=20 && \
    mkfs.vfat efiboot.img && \
    mmd -i efiboot.img efi efi/boot && \
    mcopy -vi efiboot.img $HOME/titus-live/tmp/bootx64.efi ::efi/boot/
```

### Create ISO File

```
xorriso \
    -as mkisofs \
    -iso-level 3 \
    -o "${HOME}/titus-live/titus-live.iso" \
    -full-iso9660-filenames \
    -volid "DEBIAN_CUSTOM" \
    -isohybrid-mbr /usr/lib/ISOLINUX/isohdpfx.bin \
    -eltorito-boot \
        isolinux/isolinux.bin \
        -no-emul-boot \
        -boot-load-size 4 \
        -boot-info-table \
        --eltorito-catalog isolinux/isolinux.cat \
    -eltorito-alt-boot \
        -e /EFI/boot/efiboot.img \
        -no-emul-boot \
        -isohybrid-gpt-basdat \
    -append_partition 2 0xef ${HOME}/titus-live/staging/EFI/boot/efiboot.img \
    "${HOME}/titus-live/staging"
 ```
 
 ## Source Articles
 
 - https://willhaley.com/blog/custom-debian-live-environment/
