howto_xen_omap5432uevm
======================
This file explain how to run XEN on Ti omap5432uevm board (ARM Cortex-A15).
Done by dpiotrov 19-09-2014.
Please be carefully - all the patches and configuration files in this how-to is experimental.

1. Build environment.

As environment OS used XUBUNTU 14.04.1 : 
http://nl.archive.ubuntu.com/ubuntu-cdimage-xubuntu/releases/14.04/release/xubuntu-14.04.1-desktop-amd64.iso

Installtion process:	    
sudo apt-get update
sudo apt-get upgrade
sudo apt-get build-dep qemu
sudo apt-get install gcc-4.8-arm-linux-gnueabihf g++-4.8-arm-linux-gnueabihf \
gcc-arm-linux-gnueabihf pkg-config-arm-linux-gnueabihf xapt git-core cmake \
flex bison doxygen u-boot-tools ruby lzip gawk autoconf automake gettext \
libtool subversion python-pexpect curl fakeroot tree ruby vim mc gedit \
alien autopoint

sudo xapt -a armhf -m -b libglib2.0-dev zlib1g-dev libfdt-dev \
libpixman-1-dev uuid-dev libncurses5-dev libaio-dev openssl \
libssl-dev libyajl-dev libxen-dev libtspi-dev python
sudo dpkg -i /var/lib/xapt/output/*.deb

2. U-BOOT

git clone git://git.denx.de/u-boot.git
cd u-boot/
git checkout v2014.07 -b tmp
wget -c https://raw.githubusercontent.com/eewiki/u-boot-patches/master/v2014.07/0001-omap5_common-uEnv.txt-bootz-n-fixes.patch
patch -p1 < 0001-omap5_common-uEnv.txt-bootz-n-fixes.patch
# Hypervisor mode patch required for starting XEN in HYP mode (<this how to>/u-boot/):
patch -p1 < uboot-hypervisor-mode.patch

make ARCH=arm distclean
make ARCH=arm omap5_uevm_config
make ARCH=arm

#copy u-boot parts to BOOT partition
sudo cp MLO /media/boot/
sudo cp u-boot.img /media/boot/

File  uEnv.txt  helps to run XEN with dom0 automatically:	uEnv.txt:
bootcmd=fatload mmc 0:1 0x825f0000 omap5-uevm.dtb;\
 fatload mmc 0:1 0x90000000 xen-uImage;\
 fatload mmc 0:1 0xa0000000 zImage;\
 sleep 2; bootm 0x90000000 - 0x825f0000 uenvcmd=run bootcmd;

3. XEN and XEN tools:

used xen4.5rc0	
git clone git://xenbits.xen.org/xen.git
cd xen/
	
XEN:
make dist-xen XEN_TARGET_ARCH=arm32 CROSS_COMPILE=arm-linux-gnueabihf-
mkimage -A arm -T kernel -a 0x80200000 -e 0x80200000 -C none -d "xen/xen" xen-uImage
	
XEN Tools:
edit  in file ./configure - change line  "cross_compiling=no"  to  cross_compiling=yes
./configure --host=arm-linux-gnueabihf --prefix=/usr
make dist-tools XEN_TARGET_ARCH=arm32 CROSS_COMPILE=arm-linux-gnueabihf-


4. Linux kernels:

wget https://www.kernel.org/pub/linux/kernel/v3.x/linux-3.17.4.tar.xz 
tar -xvf linux-3.17.4.tar.xz 
cd linux-3.17-rc7

DOM-0:
Copy *.dts and *.dtsi files from <this how to>/dom-0/ directory to arc/arm/boot/dts/
Copy dom0-omap5432-uevm.config from <this how to>/dom-0/ to ./.config

CROSS_COMPILE=arm-linux-gnueabihf- make ARCH=arm silentoldconfig
CROSS_COMPILE=arm-linux-gnueabihf- make ARCH=arm zImage
CROSS_COMPILE=arm-linux-gnueabihf- make ARCH=arm zImage dtbs

get zImage from arch/arm/boot/
get omap5-uevm.dtb from arch/arm/boot/dts/ 

DOM-U
Apply patches from <this how to>/dom-u/
Copy domU-omap5432-uevm.config from <this how to>/dom-u/ to ./.config

CROSS_COMPILE=arm-linux-gnueabihf- make ARCH=arm silentoldconfig
CROSS_COMPILE=arm-linux-gnueabihf- make ARCH=arm zImage

get zImage from arch/arm/boot/

5. Rootfs images:

Building of rootfs is out of scope here. Nothing specific.
You can get any ready rootfs for Cortex-A15 CPU from internet 
or prepare it by hand based on busybox.
	
Prepare ramdisk for VM's:
RDSIZE=102400 #/*100 MB*/
BLKSIZE=1024
dd if=/dev/zero of=rootfs1.img bs=$BLKSIZE count=$RDSIZE
/sbin/mke2fs -F -m 0 -b $BLKSIZE rootfs1.img $RDSIZE
mount rootfs1.img /mnt/tmpd -t ext2 -o loop=/dev/loop0
cd some_rootfs/
cp -rf * /mnt/tmpd/
cd -
umount /mnt/tmpd
cat rootfs1.img |gzip > initramfs.igz

6. Startup the system:

boot partition on micro SD card should be formatted as fat-32.
File content of boot partition:

MLO /*u-boot first stage*/
omap5-uevm.dtb /*device tree and xen boot commands*/
u-boot.img /*u-boot second stage body image*/
uEnv.txt /*as explained in u-boot part*/
xen-uImage /*XEN hypervisor image*/
zImage /*Dom-0 kernel image*/

rootfs of Dom-0 should be at second partition (after boot) formatted as ext4.
If you want to change boot configuration - please correct uEnv.txt and 
omap5-uevm-chosen.dts file.

If all the things is done right XEN and dom-0 will starting.

7. Virtual machine startup and configuration:

VM's starting by xl command fron xen tools:
xl -vvv domain-u-config.txt /* for immediate console "-c" option can be added */

VM config for rootfs on MicroSD partition:
kernel = "/usr/domus/zImage3162u7"
memory = 256
name = "vm3"
vcpus = 2
vif = [ 'mac=DE:AD:BE:AA:0F:06, bridge=xenbr0' ]
disk = [ 'phy:/dev/mmcblk0,xvda,w' ]
sdl = 1
extra = "console=hvc0 root=ca03 rw rootwait fixrtc panic=3 ip=192.168.0.93::192.168.0.1:255.255.255.0:vm3:eth0"
on_reboot = 'destroy'
on_crash = 'destroy'
on_poweroff = 'destroy'

VM config for rootfs on ramdisk:
kernel = "/usr/domus/zImage3162u7"
ramdisk = "/usr/domus/rfs2.igz"
memory = 384
name = "vm4"
vcpus = 2
vif = [ 'mac=DE:AD:BE:AA:0F:04, bridge=xenbr0' ]
sdl = 1
extra = "console=hvc0 root=/dev/ram0 ramdisk_start=0x48000000 ramdisk_size=0x5F00000 fixrtc panic=3 ip=192.168.0.94::192.168.0.1:255.255.255.0:vm4:eth0"
on_reboot = 'destroy'
on_crash = 'destroy'
on_poweroff = 'destroy'


8. Additional notes:

a. Graphic subsystem drivers was out of scope here.












