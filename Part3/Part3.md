# HD9901 Rk3399 Board移植笔记

## Part3: 提取原始配置

#### 一、解包固件
原始的`Update.img`为所谓的`release_update`镜像，需要使用`img_unpack`进行第一次解包：
```
$ ./img_unpack update.img img
rom version: 6.0.1
build time: 2020-01-18 14:13:54
chip: 33333043
checking md5sum....OK
```
再使用`afptool`解包：
```
$ ./afptool -unpack ./img update
Check file...OK
------- UNPACK -------
package-file    0x00000800      0x000002BA
Image/MiniLoaderAll.bin 0x00001000      0x0004014E
Image/parameter.txt     0x00041800      0x0000038E
Image/trust.img 0x00042000      0x00400000
Image/uboot.img 0x00442800      0x00400000
Image/misc.img  0x00843000      0x0000C000
Image/resource.img      0x0084F800      0x0014C200
Image/kernel.img        0x0099C000      0x011EA814
Image/boot.img  0x01B87000      0x01524000
Image/recovery.img      0x030AB800      0x01B14000
Image/system.img        0x04BC0000      0x47695564
RESERVED        0x00000000      0x00000000
update-script   0x4C255800      0x000003A5
recover-script  0x4C256000      0x0000010A
UnPack OK!
```
#### 二、开启adb调试
原始固件未开启`adb`授权，所以需要修改`build.prop`。
使用`simg2img`转换`system.img`：
`$ simg2img system.img system.ext4`
因为要修改`system.img`的内容，所以需要增加一点容量：
```
$ dd if=/dev/zero bs=1M count=10 >> system.ext4
输入了 10+0 块记录
输出了 10+0 块记录
10485760 字节 (10 MB, 10 MiB) 已复制，0.00853426 s，1.2 GB/s

$ e2fsck -f system.ext4 
e2fsck 1.47.0 (5-Feb-2023)
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information
Padding at end of inode bitmap is not set. Fix<y>? yes

system: ***** FILE SYSTEM WAS MODIFIED *****
system: 2893/96768 files (0.0% non-contiguous), 289198/387072 blocks

$ resize2fs system.ext4 
resize2fs 1.47.0 (5-Feb-2023)
Resizing the filesystem on system.ext4 to 395776 (4k) blocks.
The filesystem on system.ext4 is now 395776 (4k) blocks long.

$ fdisk -lu system.ext4 
Disk system.ext4: 1.51 GiB, 1621098496 bytes, 3166208 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```
挂载`system.ext4`：
`$ sudo mount system.ext4 system`
`build.prop`当中添加开启`adb`调试相关内容：
```
persist.sys.usb.config=mtp,adb
persist.service.adb.enable=1
service.adb.tcp.port=5555
persist.service.debuggable=1
ro.debuggable=1
ro.secure=0
ro.adb.secure=0
```
卸载挂载后，再次执行：
`$ e2fsck -f system.ext4`
转换`img`回`simg`：
```
$ img2simg system.ext4 system.img
```
按照`parameter.txt`内的的地址用Rockchip开发工具烧录进`system`分区，重启即可连接adb：
#### 三、提取gpio
`adb shell`下执行
```
$ cat /sys/kernel/debug/gpio
```
即可获得机器的`gpio`信息。
#### 四、提取dtb
使用解包出来的`boot.img`，具体方法详见Part1，此处给出需要的命令：
```
$ ./unmkbootimg './boot.img'
$ ./resource_tool --unpack --image=second.gz
$ dtc -I dtb -O dts -o out/rk-kernel.dts out/rk-kernel.dtb 
```