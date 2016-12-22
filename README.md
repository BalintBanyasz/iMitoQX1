# iMito QX1
Unofficial Android/Linux support for iMito QX1

### Installing Rockchip utils (rkutils)
```
mkdir rockchip
cd rockchip
git clone https://github.com/naobsd/rkutils.git
cd rkutils
for f in *.c; do gcc $f -o ${f%.*}; done
rkutils="$(pwd)"
cat <<EOF >>  ~/.bashrc
export PATH="$rkutils:$PATH"
EOF
source ~/.bashrc
```

### Installing bluetooth (RDA 5876)
Install dependencies:
```
sudo apt-get install libusb-dev libdbus-1-dev libglib2.0-dev automake libudev-dev libical-dev libreadline-dev build-essential libtool checkinstall
```

Install Bluez with RDA support:
```
git clone https://github.com/BalintBanyasz/bluez-rda.git bluez-rda
cd bluez-rda
libtoolize
aclocal
autoconf
autoheader
automake --add-missing
./configure --prefix=/usr --mandir=/usr/share/man --sysconfdir=/etc --localstatedir=/var --libexecdir=/lib
make
sudo checkinstall # Change version from rda to 4.101
sudo dpkg -i bluez_4.101-1_armhf.deb
sudo apt-mark hold bluez
```

Unfortunately, bluetooth.service dies at boot when started automatically:
```
#sudo systemctl enable bluetooth
#sudo reboot
#sudo systemctl status bluetooth
● bluetooth.service - Bluetooth service
 Loaded: loaded (/lib/systemd/system/bluetooth.service; enabled)
 Active: inactive (dead)
```

Add the following lines to rc.local before the line 'exit 0' to activate bluetooth on startup:
```
sudo nano /etc/rc.local

# Start bluetooth service and attach rda-bt
systemctl start bluetooth.service
/usr/sbin/hciattach -s 115200 /dev/ttyS0 rda 1500000 flow
```

Install Blueman:
```
sudo apt-get install python-dbus obex-data-server
wget https://github.com/radxa/apt/raw/gh-pages/rabian-stable/pool/main/b/blueman/blueman_1.23-git201403102151-1ubuntu1_armhf.deb
sudo dpkg -i blueman_1.23-git201403102151-1ubuntu1_armhf.deb
sudo apt-mark hold blueman
```

Uncomment the line 'NotShowIn=LXDE' in /etc/xdg/autostart/blueman.desktop.

### Installing Mali GPU drivers
Copy drm.ko, mali_drm.ko, ump.ko and mali.ko into /lib/modules/3.0.36+
```
sudo depmod -a 3.0.36+
```
Append the following lines to the end of /etc/modules:
```
drm
mali_drm
ump
mali
```
Create an udev rule to fix permissions:
```
sudo nano /etc/udev/rules.d/99-mali.rules
KERNEL=="mali",SUBSYSTEM=="misc",MODE="0777",GROUP="video"
KERNEL=="ump",SUBSYSTEM=="ump",MODE="0777",GROUP="video"
KERNEL=="card0",SUBSYSTEM=="drm",MODE="0666"
```

####Installing the UMP (Unified Memory Provider) userspace library
Install prequisites
```
sudo apt-get install git build-essential autoconf libtool
```
Clone the repo
```
git clone https://github.com/linux-sunxi/libump.git
cd libump
```
Build package
```
sudo apt-get install debhelper dh-autoreconf fakeroot pkg-config
dpkg-buildpackage -b
sudo dpkg -i ../libump_*.deb
```
####Installing the Mali userspace driver
Install prequisites
```
sudo apt-get install git build-essential autoconf automake
```
Clone the repo
```
git clone --recursive https://github.com/linux-sunxi/sunxi-mali.git
cd sunxi-mali
```
Configure
```
make config
```
The Mali kernel modules must be loaded, otherwise the correct settings won't be auto-detected. Make sure that config returns the following:
```
ABI="armhf" (Detected)
VERSION="r3p2-01rel1" (Detected)
EGL_TYPE="x11" (Detected)
echo "MALI_VERSION ?= r3p2-01rel1" > config.mk
echo "MALI_LIBS_ABI ?= armhf" >> config.mk
echo "MALI_EGL_TYPE ?= x11" >> config.mk
make[2]: Leaving directory '/home/debian/src/sunxi-mali'
make[1]: Leaving directory '/home/debian/src/sunxi-mali'
```

Install
```
sudo make install
```
####Installing xf86-video-fbturbo
Install prequisites
```
sudo apt-get install git build-essential xorg-dev xutils-dev x11proto-dri2-dev
sudo apt-get install libltdl-dev libtool automake libdrm-dev
```
Get the sources of xf86-video-fbturbo
```
git clone  -b mali-r3p2-support --single-branch https://github.com/ssvb/xf86-video-fbturbo.git
cd xf86-video-fbturbo
```
Compile by running:
```
autoreconf -vi
./configure --prefix=/usr
```
Make sure that configure returns the following:
```
checking ump/ump.h usability... yes
checking ump/ump.h presence... yes
checking for ump/ump.h... yes
checking for ump_open in -lUMP... yes
checking for ump_cache_operations_control in -lUMP... yes
```
If checking ump fails, run the following commands and try again:
```
sudo apt-get install libtool-bin
sudo libtool --finish /usr/lib/arm-linux-gnueabihf
```

Make:
```
make
```
Then install:
```
sudo make install
```
Setup xorg.conf:
Backup existing file if you have machybris installed:
```
sudo mv /etc/X11/xorg.conf /etc/X11/xorg.conf.libhybris
```

```
sudo cp xorg.conf /etc/X11/xorg.conf.mali
sudo rm /etc/X11/xorg.conf
sudo ln -s /etc/X11/xorg.conf.mali /etc/X11/xorg.conf
```

Edit Mali xorg.conf:
```
sudo nano /etc/X11/xorg.conf.mali
```
Add the line ```Option "AccelMethod" "CPU"``` to the "Device" section. Otherwise you may experience screen refresh issues when moving windows.


Move the mesa-egl aside:
```
sudo mv /usr/lib/arm-linux-gnueabihf/mesa-egl/ /usr/lib/arm-linux-gnueabihf/.mesa-egl/
```
Move the lima drivers out of the search path:
```
sudo mkdir /usr/lib/arm-linux-gnueabihf/bak
sudo mv /usr/lib/arm-linux-gnueabihf/libGLES* /usr/lib/arm-linux-gnueabihf/bak
sudo mv /usr/lib/arm-linux-gnueabihf/libEGL* /usr/lib/arm-linux-gnueabihf/bak
```

### 1) Building a linux kernel for the iMito QX1

**Install dependencies:**
```
sudo apt-get install build-essential lzop libncurses5-dev libssl-dev libc6:i386 lib32z1 lib32stdc++6
```

**Install the toolchain:**
```
mkdir toolchain
cd toolchain
wget https://releases.linaro.org/archive/13.04/components/toolchain/binaries/gcc-linaro-arm-linux-gnueabihf-4.7-2013.04-20130415_linux.tar.bz2
tar xvf gcc-linaro-arm-linux-gnueabihf-4.7-2013.04-20130415_linux.tar.bz2
rm gcc-linaro-arm-linux-gnueabihf-4.7-2013.04-20130415_linux.tar.bz2
toolchain="$(pwd)/gcc-linaro-arm-linux-gnueabihf-4.7-2013.04-20130415_linux/bin"
cat <<EOF >>  ~/.bashrc
export PATH="$toolchain:$PATH"
EOF
source ~/.bashrc
```

**Clone kernel sources:**
```
git clone https://github.com/BalintBanyasz/kernel_rockchip.git --branch rockchip-3.0-rbox-kk --single-branch rockchip-3.0-rbox-kk
```

**Build the kernel:**
```
cd rockchip-3.0-rbox-kk
make mrproper
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- rk3188_imito_qx1_linux_defconfig
make -j$(nproc) ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf-
 ```
**Sign the kernel image:**
```
rkcrc -k arch/arm/boot/zImage kernel.img
```
 
**Build the kernel modules:**
```
mkdir modules
make -j$(nproc) ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- INSTALL_MOD_PATH=modules modules modules_install
```

**Remove symlinks that point to files we do not need in root fs:**
```
cd modules
find . -name source | xargs rm
find . -name build | xargs rm
```

### 2) Building a boot image
```
git clone https://github.com/radxa/initrd.git
make -C initrd
```
The generated initrd.img image file will be located in the current folder.

Gzip the initrd.img file to get a compressed boot image:
```
gzip -c initrd.img > boot-unsigned.img
```

Finally, sign the image:
```
rkcrc -k boot-unsigned.img boot.img
```

### 3) Creating a Debian root filesystem image
Install the needed packages:
```
sudo apt-get install build-essential lzop libncurses5-dev libssl-dev libc6:i386 lib32z1 lib32stdc++6 debootstrap qemu-user-static
```

Clone this repo:
```
git clone https://github.com/BalintBanyasz/iMitoQX1
```

Define some variables:
```
rfs=rootfs.img
targetdir=/mnt
distro=jessie
modules=~/rockchip-3.0-rbox-kk/modules/lib/modules
```

Create the rootfs image:
```
dd if=/dev/zero of=$rfs bs=1M count=3600
mkfs.ext4 -F -L linuxroot $rfs
sudo mount -o loop $rfs $targetdir
```

Build first stage:
```
sudo debootstrap --arch=armhf --foreign $distro $targetdir
```

Copy kernel modules:
```
sudo mkdir -p $targetdir/lib/modules
sudo cp -r $modules/. $targetdir/lib/modules/
sudo cp -r iMitoQX1/modules/. $targetdir/lib/modules/3.0.36+/
```

Copy needed files from host to the target:
```
sudo cp /usr/bin/qemu-arm-static $targetdir/usr/bin/
sudo cp /etc/resolv.conf $targetdir/etc
```

Mount filesystems:
```
sudo mount -t proc proc $targetdir/proc
sudo mount -t sysfs sysfs $targetdir/sys
sudo mount -o bind /dev $targetdir/dev
sudo mount -t devpts devpts $targetdir/dev/pts
```

We should now have a minimal Debian Rootfs.

Login into the new filesystem:
```
sudo chroot $targetdir
```

Inside the chroot we need to set up the environment again:
```
distro=jessie
export LANG=C
```

Now we would run the setup for the second stage of debootstrap needs to run install the packages downloaded earlier
```
/debootstrap/debootstrap --second-stage
```

Once the package installation has finished, setup some support files and apt configuration.
```
cat <<EOT > /etc/apt/sources.list
deb http://ftp.uk.debian.org/debian $distro main contrib non-free
deb-src http://ftp.uk.debian.org/debian $distro main contrib non-free
deb http://ftp.uk.debian.org/debian $distro-updates main contrib non-free
deb-src http://ftp.uk.debian.org/debian $distro-updates main contrib non-free
deb http://security.debian.org/debian-security $distro/updates main contrib non-free
deb-src http://security.debian.org/debian-security $distro/updates main contrib non-free
EOT
```

Update Debian package database:
```
apt-get update
```

Set up locales dpkg scripts since it tends to complain otherwise; note in Debian Jessie you will also need to install the dialog package as well:
```
apt-get install locales dialog
dpkg-reconfigure locales
```
en_US.UTF-8 [*]

Fix permission for sudo:
```
chmod u+s /usr/bin/sudo
```

Install some useful packages inside the chroot:
```
apt-get install sudo openssh-server ntpdate less nano wireless-tools wpasupplicant
```

Set a root password so you can login:
```
passwd
```

Set the hostname
```
echo iMito-QX1 > /etc/hostname
nano /etc/hosts
```
Add a new line with the address 127.0.1.1 and your hostname:
```
127.0.1.1       iMito-QX1
```

Deploy kernel modules:
```
depmod -a 3.0.36+
```

Load wifi at boot:
```
echo wlan >> /etc/modules
```

####Installing X
You cannot run X as root (actually it's possible, but it's a bad practice), so you have to add new user to run X.
<pre>adduser <b>debian</b></pre>
<pre>adduser <b>debian</b> sudo</pre>

Install Xorg, LightDM and LXDE:
```
apt-get install xorg lightdm lxde
```

Modify lightdm configuration to enable autologin:
<pre>
nano /etc/lightdm/lightdm.conf
autologin-user=<b>debian</b>
autologin-user-timeout=0
</pre>

The RTL8189ES WiFi adapter seems to work better with NetworkManager, so let's replace Wicd with it:
```
apt-get purge --auto-remove wicd wicd-daemon
apt-get install network-manager-gnome
```

####Installing machybris and mackodi
Download and install machybris:
```
wget http://mac-l1.com/v0.1.2/machybris-rk3188_4.4.2-0.1.2_armhf.deb
sudo dpkg -i --force-all,confnew machybris-rk3188_4.4.2-0.1.2_armhf.deb
sudo apt-get install -f
```
Download and install mackodi:
```
wget http://mac-l1.com/v0.1.1/mackodi_isengard-jessie-0.1.1_armhf.deb
sudo dpkg -i --force-all,confnew mackodi_isengard-jessie-0.1.1_armhf.deb
sudo apt-get install -f
```

Apply sound fix:
```
wget http://mac-l1.com/v0.1.1/fix_sound.sh
sudo chmod +x fix_sound.sh
sudo ./fix_sound.sh
```

We are done inside the chroot, so quit the chroot shell
```
exit
```

Tidy up the support files
```
sudo rm $targetdir/etc/resolv.conf
sudo rm $targetdir/usr/bin/qemu-arm-static
```

Unmount filesystems (reboot if can’t):
```
sudo umount $targetdir/{proc,sys,dev/pts,dev,}
```

###4) Creating parameter.img
Create parameter file:

a) With framebuffer console enabled
```
cat <<EOT > parameter
FIRMWARE_VER:4.2.2
MACHINE_MODEL:radxa_rock
MACHINE_ID:007
MANUFACTURER:RADXA
MAGIC: 0x5041524B
ATAG: 0x60000800
MACHINE: 3066
CHECK_MASK: 0x80
KERNEL_IMG: 0x60408000
#RECOVER_KEY: 1,1,0,20,0
CMDLINE:console=ttyFIQ0 console=tty0 root=/dev/mmcblk0p1 rw rootfstype=ext4 rootdelay=10 init=/sbin/init initrd=0x62000000,0x002F0000 mtdparts=rk29xxnand:0x00002000@0x00002000(misc),0x00008000@0x00004000(kernel),0x00008000@0x0000c000(boot)
EOT
```

b) With framebuffer console disabled (required for machybris)
```
cat <<EOT > parameter
FIRMWARE_VER:4.2.2
MACHINE_MODEL:radxa_rock
MACHINE_ID:007
MANUFACTURER:RADXA
MAGIC: 0x5041524B
ATAG: 0x60000800
MACHINE: 3066
CHECK_MASK: 0x80
KERNEL_IMG: 0x60408000
#RECOVER_KEY: 1,1,0,20,0
CMDLINE:fbcon=vc:64-63 console=ttyFIQ0 console=tty0 root=/dev/mmcblk0p1 rw rootfstype=ext4 rootdelay=10 init=/sbin/init initrd=0x62000000,0x002F0000 mtdparts=rk29xxnand:0x00002000@0x00002000(misc),0x00008000@0x00004000(kernel),0x00008000@0x0000c000(boot)
EOT
```

Sign the parameter file:
```
rkcrc -p parameter parameter.img
```


###5) Creating a bootable SD card image

Image layout:

| Offset         | Image file             | Description        |
|:---------------|:-----------------------|:-------------------|
| 0x0000         | sdboot_rk3188.img      | Rockhip loader     |
| 0x2000         | parameter.img          | parameter image    |
| 0x2000+0x4000  | kernel.img             | kernel image       |
| 0x2000+0xc000  | boot.img               | boot (initrd) image|
| 0x2000+0x12001 | rootfs.img             | root filesystem    |

```
sudo e2fsck -f rootfs.img
dd if=/dev/zero of=sd.img bs=512 count=$((0x2000+0x12000+$(stat -c '%s' rootfs.img)*2/1024))
dd if=sdboot_rk3188.img of=sd.img conv=sync,fsync
dd if=parameter.img of=sd.img conv=sync,fsync seek=$((0x2000))
dd if=kernel.img of=sd.img conv=sync,fsync seek=$((0x2000+0x4000))
dd if=boot.img of=sd.img conv=sync,fsync seek=$((0x2000+0xc000))
dd if=rootfs.img  of=sd.img conv=sync,fsync seek=$((0x2000+0x12001))
```

Format image:
```
fdisk sd.img << EOF
n
p
1
81921
 
w
EOF
```

###6) Writing SD image to card
Find your sd card:
```
sudo fdisk -l
```
Write image to card:
```
sudo dd if=sd.img of=/dev/sdc bs=2M
sudo sync
```
