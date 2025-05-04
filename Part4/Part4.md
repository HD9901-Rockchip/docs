# HD9901 Rk3399 Board移植笔记

## Part4: 构建Rootfs

#### 一、概述
Part2中已介绍过Rockchip的系统分区组成，在采用独立的`boot`分区的情况下，`Rootfs`的组成包含以下3个部分:
1.基本的系统文件
2.内核模块
3.必要的设备固件
#### 二、Debian
对于Debian，我们可以使用`debootstrap`轻松完成基础系统的构建。
1.安装必要的软件包:
```
apt install -y qemu-user-static debootstrap arch-install-scripts 
```
2.使用`debootstrap`构建基础系统:
```
mkdir rootfs
sudo debootstrap --arch=arm64 --foreign bookworm rootfs/ http://mirrors.ustc.edu.cn/debian/
```
3.chroot进入构建的rootfs:
```
sudo arch-chroot rootfs
```
4.执行第二阶段:
```
/debootstrap/debootstrap --second-stage
```
5.修改主机名:
```
echo debian >/etc/hostname
```
6.添加hosts:
```
cat >/etc/hosts <<EOF
127.0.0.1       localhost
127.0.1.1       debian
127.0.1.2       localhost.localdomain

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
EOF
```
7.配置dns:
```
rm -rf /etc/resolv.conf
echo "search lan" >/etc/resolv.conf
echo "nameserver 114.114.114.114" >>/etc/resolv.conf
cat /etc/resolv.conf
```
8.配置更新源:
```
sed -i 's/deb.debian.org/mirrors.ustc.edu.cn/g' /etc/apt/sources.list
```
9.rootfs安装必要的软件包:
```
apt update
apt install bash-completion htop nano vim curl wget axel unar libdrm-dev network-manager wireless-tools iw bluez bluez-tools rfkill pciutils usbutils alsa-utils lshw ssh openssh-server initramfs-tools
```
10.允许root用户ssh登录:
```
echo 'PermitRootLogin yes' >> /etc/ssh/sshd_config
```
11.修改root密码:
```
passwd root
```
12.清理缓存:
```
cat /dev/null > ~/.bash_history && history -c
```
13.退出chroot:
```
exit
```
14.安装内核模块:
编译内核时，设置为`M`的驱动会被编译生成`.ko`格式的内核模块，我们需要在内核编译完成的目录执行

```
sudo make ARCH=arm64 O=out CROSS_COMPILE=aarch64-linux-gnu-  INSTALL_MOD_PATH='<rootfs文件夹的路径>' modules_install
```
注:对于debian系linux系统，我们在编译内核的时候可以使用`make bindeb-pkg`生成内核的deb安装文件（kernel、header、libc），直接拷贝进`tmp`目录使用`dpkg`安装，会自动在`/boot`路径下生成`config`、`vmlinuz`、`initrd`，并安装内核模块和libc。

15.安装ap6256固件:
我们需要将`brcmfmac43455-sdio.bin`、`brcmfmac43455-sdio.vamrs,rock960.txt`拷贝进