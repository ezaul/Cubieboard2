# Cubieboard2
How to compile for Cubieboard2 in 2012


# Cluster #
Propagar comando
for a in $(echo {2..7}); do ssh cubie$a "echo cpu0 > /sys/class/leds/green\:ph20\:led1/trigger &"; done


# Add sources.list
deb http://www.emdebian.org/debian/ wheezy main

deb http://www.emdebian.org/debian/ sid main
deb http://packages.cubian.org/ wheezy main

wget -qO - http://packages.cubian.org/cubian.gpg.key | apt-key add -

# APT-GET
apt-get install emdebian-archive-keyring
apt-get install u-boot-tools

apt-get install debootstrap qemu-user-static binfmt-support qemu qemu-launcher qtemu qemu-kvm qemu-kvm-dbg qemu-keymaps qemu-system qemu-user qemu-user-static qemu-utils kvm aqemu

apt-get install dhcp3-client udev netbase ifupdown iproute openssh-server iputils-ping wget net-tools ntpdate ntp vim nano less tzdata console-tools module-init-tools mc

apt-get install module-assistant build-essential bin86 binutils bison fakeroot flex libncurses5 libncurses5-dev
apt-get install git git-core libz-dev libssl-dev libcurl4-gnutls-dev libexpat1-dev gettext lzma p7zip-full

apt-get install gcc-4.7-arm-linux-gnueabihf cpp-4.7-arm-linux-gnueabihf g++-4.7-arm-linux-gnueabihf gcc-4.7-arm-linux-gnueabihf gcc-4.7-arm-linux-gnueabihf-base gcc-4.7-plugin-dev-arm-linux-gnueabihf gccgo-4.7-arm-linux-gnueabihf gfortran-4.7-arm-linux-gnueabihf gobjc++-4.7-arm-linux-gnueabihf gobjc-4.7-arm-linux-gnueabihf

# Instrução de link compilador
for i in /usr/bin/arm-linux-gnueabihf*-4.7 ; do j=${i##/usr/bin/}; ln -s $i ${j%%-4.7} ; done
for i in arm-linux-gnueabihf-*-4.7; do ln -s $i ${i%%-4.7}; done

# Gits - Git Util

git clone git://github.com/linux-sunxi/u-boot-sunxi.git
./mkconfig arch=arm cpu=armv7 board=Cubieboard2 soc=sunxi
./mkconfig ARCH=arm CPU=armv7 SOC=sunxi BOARD=Cubieboard2

make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- Cubieboard2_config 
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- 


git clone git://github.com/linux-sunxi/sunxi-tools.git
make misc
./sunxi-nand-image-builder -p 8192 -o 640 -e 0x100000 -b -u 4096 -c 64/1024 sunxi-spl.bin uboot.bin
nandwrite -o -n /dev/mtd0 uboot.bin
nandwrite -o -n /dev/mtd1 uboot.bin
nandwrite -p /dev/mtd2 u-boot-dtb.bin
nandwrite -p /dev/mtd3 u-boot-dtb.bin

git clone git://github.com/linux-sunxi/sunxi-boards.git

# Git Kernel
git clone git://github.com/linux-sunxi/linux-sunxi.git -b sunxi-devel linux-sunxi-3.12
git clone git://github.com/linux-sunxi/linux-sunxi.git -b experimental/sunxi-3.10 linux-sunxi-3.10

# Git XEN-ARM
#Repositorio do sun7i_xen
git clone -b sun7i_xen_domU https://github.com/bjzhang/linux-allwinner.git

# Compilando KERNEL

ARCH=arm make multi_v7_defconfig
make ARCH=arm distclean
make ARCH=arm clean
make distclean
make clean

# Esse é pra ver o menu pra navegar nos recursos do kernel#
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- menuconfig

# Compila kernel e os módulos#
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- LOADADDR=0x40008000 uImage modules

# Monta o cartão sd na /mnt e daí roda esse aí pra instalar os módulos na /mnt/lib #
# mas o INSTALL_MOD_PATH=/mnt mesmo, não /mnt/lib #
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- INSTALL_MOD_PATH=/mnt modules_install

# Dicas do DEFCONFIG 
os defconfig tem que ir catando
aí copia pra dentro do fork do kernel como .config
lembre de habilitar a segunda opção de suporte a carregamento de módulos do kernel, senão vc só trabalha com módulos internos, não consegue fazer módulos externos e trabalhar com o modprobe

# Compilando 2
#Documentos http://linux-sunxi.org/Mainline_Kernel_Howto

# Compila Kernel e dtbs
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- multi_v7_defconfig
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- LOADADDR=0x40008000 uImage modules dtbs
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- sun7i-a20-cubieboard2.dtb
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- INSTALL_MOD_PATH=/mnt modules_install

# Particion

fdisk /dev/sdb
d
n
p
1
enter
enter
w
e2fsck -f /dev/sdb1
resize2fs /dev/sdb1

# Apaga MBR 
dd if=/dev/zero of=/dev/sdb bs=1024 seek=544 count=128

# Nova
dd if=u-boot-sunxi-with-spl.bin of=/dev/sdb bs=1024 seek=8

# Antiga
dd if=spl/sunxi-spl.bin of=/dev/sdb bs=1024 seek=8
dd if=u-boot.bin of=/dev/sdb bs=1024 seek=32

# Criar um BOOT.CMD - BOOT.SCR##

vi boot.cmd
setenv bootargs console=tty0 console=ttyS0,115200 hdmi.audio=EDID:0 disp.screen0_output_mode=EDID:1280x800p60 root=/dev/mmcblk0p1 rootfstype=ext4 rootwait panic=10
ext4load mmc 0 0x46000000 boot/uImage
ext4load mmc 0 0x49000000 boot/cubie2.dtb
env set fdt_high ffffffff
bootm 0x46000000 - 0x49000000

# Opção 2
vi boot.cmd
setenv bootargs console=tty0 console=ttyS0,115200 hdmi.audio=EDID:0 disp.screen0_output_mode=EDID:1280x800p60 root=/dev/mmcblk0p1 rootwait panic=10
fatload mmc 0 0x48000000 uImage
fatload mmc 0 0x49000000 cubie2.dtb
env set fdt_high ffffffff
bootm 0x48000000 - 0x49000000

# Opção 3
setenv bootargs console=tty0 console=ttyS0,115200 hdmi.audio=EDID:0 disp.screen0_output_mode=EDID:1280x800p60 root=/dev/sda1 rootfstype=ext4 rootwait panic=10
scsi scan
ext4load scsi 0 0x46000000 boot/uImage
ext4load scsi 0 0x49000000 boot/cubie2.dtb
env set fdt_high ffffffff
bootm 0x46000000 - 0x49000000

# Opção 4
setenv bootargs console=tty0 console=ttyS0,115200 hdmi.audio=EDID:0 disp.screen0_output_mode=EDID:1280x800p60 root=/dev/sda1 rootfstype=ext4 rootwait panic=10
scsi scan
ext4load scsi 0 0x42000000 boot/uInitrd
ext4load scsi 0 0x46000000 boot/zImage
ext4load scsi 0 0x49000000 boot/sun7i-a20-cubieboard2.dtb
env set fdt_high ffffffff
bootz 0x46000000 0x42000000 0x49000000

mkimage -C none -A arm -T script -d /boot/boot.cmd /boot/boot.scr

chmod 755 boot.scr

# Criar ROOTFS##

mkdir chroot
cd chroot
debootstrap --foreign --arch armhf wheezy .
cp /usr/bin/qemu-arm-static usr/bin
LC_ALL=C LANGUAGE=C LANG=C chroot . /debootstrap/debootstrap --second-stage
LC_ALL=C LANGUAGE=C LANG=C chroot . dpkg --configure -a
chroot . passwd
echo Cubie > etc/hostname
cp /etc/resolv.conf etc
echo "deb http://ftp.br.debian.org/ wheezy main contrib non-free" > etc/apt/sources.list
echo "deb http://security.debian.org/ wheezy main contrib non-free" >> etc/apt/sources.list
chroot . apt-get update
chroot . apt-get dist-upgrade
chroot . apt-get install nvi
echo "T0:2345:respawn:/sbin/getty -L ttyS0 115200 vt100" >> etc/inittab

# Copiar uImage para o diretorio /boot desse rootfs

usado somente se tiver compilado com MODULOS

make C <dir do kernel> INSTALL_MOD_PATH=$(pwd) ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf modules_install

tar --exclude=qemu-arm-static -cf - . | tar -C /mnt -xvf -


# Aplicando PATCH
patch -p1 arch/arm/mach-sunxi/sunxi.c b/arch/arm/mach-sunxi/sunxi.c < "onde vc salvou o arquivo/reset6i.patch"

patch -Np1 -i /root/cubie/openwrt/target/linux/sunxi/patches-3.12/176-add-dt-rtc-for-sun4i-7i.patch


# #REPOSITORI RASPI #

deb http://raspbian.ufms.br/raspbian/ wheezy main
deb-src http://raspbian.ufms.br/raspbian/ wheezy main

# Raspberry Pi Foundation official repository
deb http://archive.raspberrypi.org/debian wheezy main


# Erro
#spl: not an uImage at 1600#

env set fdt_high ffffffff 
ou
saveenv set fdt_high ffffffff


# #################### Syncronizar ####################
vi diretorios.txt
/mnt/*

rsync -avc --exclude-from=diretorios.txt / /mnt

# ######################################################################
### Clonar o HD físico em /dev/sda para o outro hd físico em /dev/sdb: ##
dd if=/dev/sda of=/dev/sdb bs=4096 conv=notrunc,noerror

notrunc – diz ao dd para manter a integridade dos dados (não truncar nenhum dado)
noerror – diz ao dd para ignorar erros e continuar o processo caso encontrar algum problema

### Criar a imagem do HD de origem e armazená-lo no HD externo: ##
dd if=/dev/sda conv=sync,noerror bs=64K > /mnt/sda.img

### Para recuperar a imagem: ##
dd if=/mnt/sda.img of=/dev/sda conv=sync,noerror bs=64k
