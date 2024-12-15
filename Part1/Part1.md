# HD9901 Rk3399 Board移植笔记

## Part1：原始固件处理

#### 一、固件处理
原厂的Update.img使用AndroidTool或者RKDevTool在Windows下进行解包最方便，Linux下可以考虑使用`rkImageMaker`和`afptool`。


```
$ ./rkImageMaker -unpack './update.img' out
********RKImageMaker ver 1.63********
Unpacking image, please wait...
Exporting boot.bin
Exporting firmware.img
Unpacking image success.
```
继续解包`firmware.img`:
```
$ ./afptool -unpack ./out/firmware.img ./out/out
Android Firmware Package Tool v1.62
Check file... OK
------- UNPACK ------
package-file    0x0000000000000800      0x00000000000002BA
Image/MiniLoaderAll.bin 0x0000000000001000      0x000000000004014E
Image/parameter.txt     0x0000000000041800      0x0000000000000382
Image/trust.img 0x0000000000042000      0x0000000000400000
Image/uboot.img 0x0000000000442800      0x0000000000400000
Image/misc.img  0x0000000000843000      0x000000000000C000
Image/resource.img      0x000000000084F800      0x000000000014C200
Image/kernel.img        0x000000000099C000      0x00000000011EA814
Image/boot.img  0x0000000001B87000      0x0000000001524000
Image/recovery.img      0x00000000030AB800      0x0000000001B14000
Image/system.img        0x0000000004BC0000      0x0000000047695564
update-script   0x000000004C255800      0x00000000000003A5
recover-script  0x000000004C256000      0x000000000000010A
Unpack firmware OK!
------ OK ------
```
不仅可以得到镜像文件，也能得到分区镜像的写入地址。对于运行Linux系统来说最重要的是提取设备树，RK3399的设备树可以从`boot.img`中提取，也可以从`resource.img`中提取。

以boot.img为例，使用`unmkbootimg`解包：
```
$ ./unmkbootimg './boot.img'
unmkbootimg version 1.2 - Mikael Q Kuisma <kuisma@ping.se>
Kernel size 18786312
Kernel address 0x60408000
Ramdisk size 1968324
Ramdisk address 0x62000000
Secondary size 1360384
Secondary address 0x60f00000
Kernel tags address 0x60088000
Flash page size 16384
Board name is ""
Command line "buildvariant=user"

*** WARNING ****
This image is built using NON-standard mkbootimg!
OFF_KERNEL_ADDR is 0x00380100
OFF_RAMDISK_ADDR is 0x01F78100
OFF_SECOND_ADDR is 0x00E78100
Please modify mkbootimg.c using the above values to build your image.
****************

Extracting kernel to file zImage ...
Extracting root filesystem to file initramfs.cpio.gz ...
Extracting second to file second.gz ...
All done.
---------------
To recompile this image, use:
  mkbootimg --kernel zImage --ramdisk initramfs.cpio.gz --base 0x60087f00 --cmdline 'buildvariant=user' --pagesize 16384 -o new_boot.img
---------------

```
执行结果后面给出了使用`mkbootimg`重新打包复原的命令，解包出来的文件有:
```
$ ls
initramfs.cpio.gz
second.gz
zImage
```
其中`zImage`是Linux内核，`initramfs.cpio.gz`是ramdisk，`second.gz`内包含第一屏logo和设备树，本质上和`resource.img`是一个东西，都可以用`resource_tool`解包：
```
$ ./resource_tool --unpack --image=second.gz
Dump header:
partition version:0.0
header size:1
index tbl:
        offset:1        entry size:1    entry num:3
Dump Index table:
entry(0):
        path:rk-kernel.dtb
        offset:4        size:75985
entry(1):
        path:logo.bmp
        offset:153      size:640922
entry(2):
        path:logo_kernel.bmp
        offset:1405     size:640922
Unack second.gz to out successed!

$ ls out/
logo.bmp
logo_kernel.bmp
rk-kernel.dtb
```
对于`resource.img`：
```
$ ./resource_tool --unpack --image=resource.img
Dump header:
partition version:0.0
header size:1
index tbl:
        offset:1        entry size:1    entry num:3
Dump Index table:
entry(0):
        path:rk-kernel.dtb
        offset:4        size:75985
entry(1):
        path:logo.bmp
        offset:153      size:640922
entry(2):
        path:logo_kernel.bmp
        offset:1405     size:640922
Unack resource.img to out successed!

$ ls out/
logo.bmp
logo_kernel.bmp
rk-kernel.dtb
```
**注意**：打包时，解包出来的文件务必和`resource_tool`放在**同一目录**，否则打包结构会错误。
打包`resource.img`：
```
./resource_tool --verbose --pack ./logo.bmp ./logo_kernel.bmp ./rk-kernel.dtb resource.img
```
打包`second.gz`：
```
./resource_tool --verbose --pack --image=second.gz ./logo.bmp ./logo_kernel.bmp ./rk-kernel.dtb
```
解包出来的`rk-kernel.dtb`就是编译后的Android设备树，可以通过`dtc`还原为可读的dts文件：
```
dtc -I dtb -O dts -o out/rk-kernel.dts out/rk-kernel.dtb 
```
修改完成后可以对dts文件进行再次编译：
```
dtc -I dts -O dtb -o out/rk-kernel.dtb out/rk-kernel.dts
```

