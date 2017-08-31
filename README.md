# debian-jessie-armhf-rpi23_4.4.13-v7-_aufs4
some build guide and binary for debian-jessie-armhf-rpi23

# README

## 0. install debootstrap on ubuntu-15.04_x86_64
```sh
apt-get install debian-archive-keyring debootstrap
```

## 1. preare debian-jess-armhf basic file system

#### debootstrap basic filesystem
```
debootstrap --verbose --arch armhf --foreign jessie debian-jessie-armhf http://ftp.tw.debian.org/debian
```
#### add dns server 
```
echo "nameserver 168.95.1.1" > debian-jessie-armhf/etc/resolv.conf
```

#### add /etc/profile
```
cat > debian-jessie-armhf/etc/profile << EOF
# /etc/profile: system-wide .profile file for the Bourne shell (sh(1))
# and Bourne compatible shells (bash(1), ksh(1), ash(1), ...).

if [ "$PS1" ]; then
  if [ "$BASH" ] && [ "$BASH" != "/bin/sh" ]; then
    # The file bash.bashrc already sets the default PS1.
    # PS1='\h:\w\$ '
    if [ -f /etc/bash.bashrc ]; then
      . /etc/bash.bashrc
    fi
  else
    if [ "`id -u`" -eq 0 ]; then
      PS1='# '
    else
      PS1='$ '
    fi
  fi
fi

# The default umask is now handled by pam_umask.
# See pam_umask(8) and /etc/login.defs.

if [ -d /etc/profile.d ]; then
  for i in /etc/profile.d/*.sh; do
    if [ -r $i ]; then
      . $i
    fi
  done
  unset i
fi

alias ls='ls --color=auto'
export PATH=$PATH:/sbin:/bin:/usr/bin:/usr/sbin
EOF

```

#### add chroot script
```
cat > debian-jessie-armhf/ch-mount.sh << EOF
#!/bin/bash

function mnt() {
    sudo mount -t proc none ${2}proc
    sudo mount -o bind /dev ${2}dev
    sudo mount -o bind /dev/pts ${2}dev/pts
    sudo mount -t sysfs sys ${2}sys
    sudo chroot ${2} /bin/bash
}

function umnt() {
    sudo umount ${2}proc
    sudo umount ${2}sys
    sudo umount ${2}dev/pts
    sudo umount ${2}dev

}

if [ "$1" == "-m" ] && [ -n "$2" ] ;
then
    umnt $1 $2
    mnt $1 $2
elif [ "$1" == "-u" ] && [ -n "$2" ];
then
    umnt $1 $2
else
    echo ""
    echo "Either 1'st, 2'nd or both parameters were missing"
    echo ""
    echo "1'st parameter can be one of these: -m(mount) OR -u(umount)"
    echo "2'nd parameter is the full path of rootfs directory(with trailing '/')"
    echo ""
    echo "For example: ch-mount -m /media/sdcard/"
    echo ""
    echo 1st parameter : ${1}
    echo 2nd parameter : ${2}
fi

EOF
chmod +x debian-jessie-armhf/ch-mount.sh
```

#### confgiure timezone, hostname
```
echo "jessie-pi" > debian-jessie-armhf/etc/hostname
echo "Asia/Taipei" > debian-jessie-armhf/etc/timezone
```

#### configure fstab
```
cat > debian-jessie-armhf/etc/fstab << EOF
/dev/mmcblk0p1 /boot vfat noatime 0 2
/dev/mmcblk0p2 / ext4 noatime 0 1
EOF
```

#### copy qemu-arm-static
```
cp /usr/bin/qemu-arm-static debian-jessie-armhf/usr/bin/
```

#### register qemu-arm-static as ARM interpreter to the kernel linux
```
sudo su
echo ':arm:M::\x7fELF\x01\x01\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02\x00\x28\x00:\xff\xff\xff\xff\xff\xff\xff\x00\xff\xff\xff\xff\xff\xff\xff\xff\xfe\xff\xff\xff:/usr/bin/qemu-arm-static:' > /proc/sys/fs/binfmt_misc/register
exit
```

#### chroot 
```
./debian-jessie-armhf/ch-mount.sh -m `pwd`/debian-jessie-armhf/
source /etc/profile
```

#### debootstrap second stage
```
/debootstrap/debootstrap --second-stage --verbose
```


#### configure sources.list (change mirror to your region)
```
cat > /etc/apt/sources.list << EOF
deb http://ftp.tw.debian.org/debian jessie main contrib non-free
deb http://ftp.tw.debian.org/debian jessie-updates main contrib non-free
deb http://security.debian.org jessie/updates main contrib non-free
deb http://archive.raspberrypi.org/debian jessie main
EOF
```
```
cat > /etc/apt/preferences.d/raspberrypi << EOF
Package: *
Pin: origin archive.raspberrypi.org
Pin-Priority: 1

Package: raspberrypi-bootloader
Pin: origin archive.raspberrypi.org
Pin-Priority: 1000

Package: libraspberrypi0
Pin: origin archive.raspberrypi.org
Pin-Priority: 1000

Package: libraspberrypi-bin
Pin: origin archive.raspberrypi.org
Pin-Priority: 1000
EOF
```

#### add raspberrypi.gpg.key to system
```
wget http://archive.raspberrypi.org/debian/raspberrypi.gpg.key
apt-key add raspberrypi.gpg.key
rm raspberrypi.gpg.key
```

#### update  
```
apt-get update
```



#### install rpi firmware, boot, kernel
```
apt-get install libraspberrypi-bin libraspberrypi0 raspberrypi-bootloader
```


#### configure /boot/cmdline.txt
```
cat > /boot/cmdline.txt << EOF
dwc_otg.lpm_enable=0 console=serial0,115200 console=tty1 root=/dev/mmcblk0p2 rootfstype=ext4 elevator=deadline fsck.repair=yes rootwait
EOF
```

#### configure /boot/config.txt
```
cat > /boot/config.txt << EOF
# For more options and information see
# http://www.raspberrypi.org/documentation/configuration/config-txt.md
# Some settings may impact device functionality. See link above for details

# uncomment if you get no picture on HDMI for a default "safe" mode
#hdmi_safe=1

# uncomment this if your display has a black border of unused pixels visible
# and your display can output without overscan
#disable_overscan=1

# uncomment the following to adjust overscan. Use positive numbers if console
# goes off screen, and negative if there is too much border
#overscan_left=16
#overscan_right=16
#overscan_top=16
#overscan_bottom=16

# uncomment to force a console size. By default it will be display's size minus
# overscan.
#framebuffer_width=1280
#framebuffer_height=720

# uncomment if hdmi display is not detected and composite is being output
#hdmi_force_hotplug=1

# uncomment to force a specific HDMI mode (this will force VGA)
#hdmi_group=1
#hdmi_mode=1

# uncomment to force a HDMI mode rather than DVI. This can make audio work in
# DMT (computer monitor) modes
#hdmi_drive=2

# uncomment to increase signal to HDMI, if you have interference, blanking, or
# no display
#config_hdmi_boost=4

# uncomment for composite PAL
#sdtv_mode=2

#uncomment to overclock the arm. 700 MHz is the default.
#arm_freq=800

# Uncomment some or all of these to enable the optional hardware interfaces
#dtparam=i2c_arm=on
#dtparam=i2s=on
#dtparam=spi=on

# Uncomment this to enable the lirc-rpi module
#dtoverlay=lirc-rpi

# Additional overlays and parameters are documented /boot/overlays/README

# Enable audio (loads snd_bcm2835)
dtparam=audio=on
start_x=0

# Enable PI3 UART
enable_uart=1


# Overclock PI3
#arm_freq=1300
#over_voltage=4
#temp_limit=80
#core_freq=500

EOF
```

#### set eth0 to dhcp
```
cat > /etc/network/interfaces.d/eth0 << EOF
auto eth0
iface eth0 inet dhcp
EOF
```

#### install sshd package & change root passwd & configure sshd_config
```
apt-get install openssh-server --no-install-recommends
sed -i 's/PermitRootLogin without-password/PermitRootLogin yes/g' /etc/ssh/sshd_config
systemctl enable ssh
passwd root
```

#### install necessary package
```
apt-get install dbus console-data console-common console-setup tzdata locales keyboard-configuration
```

#### configure locale
```
dpkg-reconfigure locales
locale-gen "en_US.UTF-8"
```

#### configure timesyncd & disable hwclock
```
systemctl enable systemd-timesyncd
systemctl disable hwclock-save
systemctl enable dbus
systemctl enable systemd-localed.service
```


#### exit & clean up & umount
```
apt-get clean
exit
./debian-jessie-armhf/ch-mount.sh -u `pwd`/debian-jessie-armhf/
```


## 2. Copy to sd card
Now we need to copy the above file system to SD CARD. Assuming the /dev/sdb1 is fat32, and /dev/sdb2 is ext4. 

#### mount sdb1, sdb2
```
mount /dev/sdb1 boot
mount /dev/sdb2 root
```

#### copy boot file to /dev/sdb1
```
cp -arpf debian-jessie-armhf/boot/* boot/
```
#### copy root file sytem to /dev/sdb2
```
rsync -av --exclude=/boot debian-jessie-armhf/ root/
```

#### sync & umount sd card
```
sync ; umount boot root
```


## 3. get my own debian-jessie-armhf_rpi23.tar.gz
download `debian-jessie-armhf_rpi23.tar.gz_*` from here and do the following instuctions. the default password is `1234`

#### switch to RPi file system
```
mkdir root
mount /dev/sdb2 root
mkdir root/boot
mount /dev/sdb1 root/boot
cd root
cat debian-jessie-armhf_rpi23.tar.gz_a* | tar -zxvp
sync ; umount root/boot ; umount root
```

## 4. get rpi23 kernel with aufs4 support
download `rpi23_4.4.13-v7+_aufs4.tar.gz` here, and then tar -zxvpf rpi23_4.4.13-v7+_aufs4.tar.gz from your rpi console. 
* support `aufs4`
* CPU power consumption default to `ondemand`
* support `Multi-core scheduler`
