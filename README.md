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

Add the following lines to rc.local before the line 'exit 0':
```
sudo nano /etc/rc.local

# Start bluetooth service and attach rda-bt
systemctl start bluetooth.service
/usr/sbin/hciattach -s 115200 /dev/ttyS0 rda 1500000 flow
```

Install Blueman:
```
sudo apt-get install obex-data-server
wget https://github.com/radxa/apt/raw/gh-pages/rabian-stable/pool/main/b/blueman/blueman_1.23-git201403102151-1ubuntu1_armhf.deb
sudo dpkg -i blueman_1.23-git201403102151-1ubuntu1_armhf.deb
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
wget https://releases.linaro.org/13.04/components/toolchain/binaries/gcc-linaro-arm-linux-gnueabihf-4.7-2013.04-20130415_linux.tar.bz2
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
 
# sign the kernel image
rkcrc -k arch/arm/boot/zImage kernel.img
 
# build the kernel modules
mkdir modules
make -j$(nproc) ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- INSTALL_MOD_PATH=modules modules modules_install
 
# remove symlinks that point to files we do not need in root fs
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

Gzip the initrd.img file to get a boot image:
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
sudo apt-get install build-essential lzop libncurses5-dev libssl-dev libc6:i386 lib32z1 lib32stdc++6
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
sudo dd if=/dev/zero of=$rfs bs=1M count=3600
sudo mkfs.ext4 -F -L linuxroot $rfs
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
sudo cp wlan.ko $targetdir/lib/modules/3.0.36+/
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
distro=wheezy
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

Install Xorg:
```
apt-get install xorg
```

Install LXDE:
```
apt-get install lxde
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
