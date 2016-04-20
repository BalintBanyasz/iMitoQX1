# iMito QX1
Unofficial Android/Linux support for iMito QX1

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
