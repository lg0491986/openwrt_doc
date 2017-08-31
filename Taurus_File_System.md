# Taurus文件系统的前世今生
[TOC]

## Overlay FileSystem
> OverlayFS is used to merge two filesystems, one read-only and the other writable.    

* OverlayFS通常也被成为Union-FileSystem，它构建于其他的文件系统之上，并提供了一种统一的视图。    
* 写时复制，即如果对只读文件系统中的文件进行修改，那么系统首先会在可写文件系统创建文件所在的目录，同时把需要修改的文件拷贝一份到新建的目录，然后打开，修改，保存。    

|-------|挂载点|分区类型|挂载说明|
|-------|--------|------|--------|
|/      |        |overlayfs||
|       |/rom    |只读类型文件系统|通常为rootfs分区|
|       |/overlay|可写类型文件系统|通常为rootfs_data分区|

比如运行```ls /rom```可以看到如下输出：
```bash
dir0
dir0/file0
dir1
dir1/file1
file2
file3 
```

比如运行```ls /overlay```可以看到如下输出：
```bash
dir1
dir1/file
file3
file4
```

那么运行```ls /```就可以看到如下输出：
```bash
/rom
/rom/dir0
/rom/dir0/file0
/rom/dir1
/rom/dir1/file1
/rom/file2
/rom/file3 

/overlay
/overlay/dir1
/overlay/dir1/file
/overlay/file3
/overlay/file4

dir0
dir0/file0
dir1
dir1/file1
dir1/file
file2
file3
file4
```
可以看到，在```/```下面看到的是```/rom```和```/overlay```两个目录的最小合集。
### 究竟file3的内容是/rom/file3还是/overlay/file3的呢？
> 答案是```/overlay/file3```    

## 今生
### 当前分区表
要查看当前的分区表，最好最直接的方式就是dmesg -c查看设备启动时的输出比如：
``` bash
[    1.378988] 17 ofpart partitions found on MTD device spi0.0
[    1.384477] Creating 17 MTD partitions on "spi0.0":
[    1.389326] 0x000000000000-0x000000040000 : "0:SBL1"
[    1.395502] 0x000000040000-0x000000060000 : "0:MIBIB"
[    1.400652] 0x000000060000-0x0000000c0000 : "0:QSEE"
[    1.405798] 0x0000000c0000-0x0000000d0000 : "0:CDT"
[    1.410726] 0x0000000d0000-0x0000000e0000 : "0:DDRPARAMS"
[    1.416153] 0x0000000e0000-0x0000000f0000 : "0:APPSBLENV"
[    1.421554] 0x0000000f0000-0x000000170000 : "0:APPSBL"
[    1.426680] 0x000000170000-0x000000180000 : "0:oldART"
[    1.431708] 0x000000180000-0x000000980000 : "0:HLOS"
[    1.436737] 0x000000980000-0x000002000000 : "rootfs"
[    1.441626] 0x000002000000-0x000002800000 : "0:HLOS1"
[    1.446742] 0x000002800000-0x000003c00000 : "rootfs_1"
[    1.451838] 0x000003c00000-0x000003f50000 : "rootfs_data"
[    1.457289] 0x000003f50000-0x000003fd0000 : "0:APPSBL1"
[    1.462506] 0x000003fd0000-0x000003fe0000 : "BoardInfo"
[    1.467819] 0x000003fe0000-0x000003ff0000 : "RebootCause"
[    1.473241] 0x000003ff0000-0x000004000000 : "0:ART"
[    1.489499] libphy: ipq40xx_mdio: probed
```
> *每个分区的作用，请参见《Taurus入门手册》，这里不再详述。    
> 这里需要注意，```rootfs_data```分区的起始位置(0x000003c00000)和结束地址(0x000003f50000)和其他分区并没有重叠，这个和WIA3200-80S等MTK方案是有区别的，详细的关于WIA3200-80S的分区情况会在后面WIA3200-80S一节讲到。*    

### 挂载情况
通过mount可以查看当前的系统目录挂载情况
```bash
root@OpenWrt:~# mount
rootfs on / type rootfs (rw)
mtd:rootfs on /rom type squashfs (ro,relatime)
proc on /proc type proc (rw,nosuid,nodev,noexec,noatime)
sysfs on /sys type sysfs (rw,nosuid,nodev,noexec,noatime)
tmpfs on /tmp type tmpfs (rw,nosuid,nodev,noatime)
tmpfs on /tmp/root type tmpfs (rw,noatime,mode=755)
overlayfs:/tmp/root on / type overlayfs (rw,noatime,lowerdir=/,upperdir=/tmp/root)
/dev/mtdblock12 on /etc/config type jffs2 (rw,relatime)
tmpfs on /dev type tmpfs (rw,nosuid,relatime,size=512k,mode=755)
devpts on /dev/pts type devpts (rw,nosuid,noexec,relatime,mode=600)
debugfs on /sys/kernel/debug type debugfs (rw,noatime)
/dev/mtdblock11 on /mnt/mtdblock1 type squashfs (ro,relatime)
```
|-------|挂载点|分区类型|挂载说明|
|-------|--------|------|--------|
|/      |        |overlayfs||
|       |/rom    |squashfs|mtd:rootfs 即rootfs分区|
|       |/overlay|tmpfs|/tmp/root 即内存|
|       |/etc/config|jffs2|/dev/mtdblock12 即rootfs_data分区|

* 从上面的挂载信息可以看出，```/overlay```挂载的是内存，是可以写，但是无法在掉电或者重启时保存；    
* 而```/etc/config```是***jffs2***文件系统类型，是可写的，且在掉电或者重启时能够保存其中的文件；

### 系统特点
* Squashfs用来存储系统必须的文件；
* Squashfs具有更好的压缩率，能够最大限度的节约Flash空间；
* Squashfs为只读，能够在系统发生异常时，恢复到默认状态；
* tmpfs为可写，掉电或者重启后，对文件的修改就会恢复默认值，可以防止异常修改；
* ```/etc/config```可存储配置，保证配置文件能够正常存储；

### 升级对分区的影响
#### 恢复出厂设置
升级时，如果要求恢复出厂设置，那么会直接调用```jffs2reset -y```对rootfs_data区进行擦除动作；

#### 保存配置
* 由于rootfs_data是一个独立分区，并不需要做特别的处理。也因此之前对配置的修改也会被保存下载。    

## 前世
### 过去的分区表
和当前的没有发生变更

### 挂载情况
通过mount可以查看当前的系统目录挂载情况
```bash
root@OpenWrt:~# mount
rootfs on / type rootfs (rw)
mtd:rootfs on /rom type squashfs (ro,relatime)
proc on /proc type proc (rw,nosuid,nodev,noexec,noatime)
sysfs on /sys type sysfs (rw,nosuid,nodev,noexec,noatime)
tmpfs on /tmp type tmpfs (rw,nosuid,nodev,noatime)
/dev/mtdblock12 on /overlay type jffs2 (rw,noatime)
overlayfs:/overlay on / type overlayfs (rw,noatime,lowerdir=/,upperdir=/overlay)
tmpfs on /dev type tmpfs (rw,nosuid,relatime,size=512k,mode=755)
devpts on /dev/pts type devpts (rw,nosuid,noexec,relatime,mode=600)
debugfs on /sys/kernel/debug type debugfs (rw,noatime)
```
|-------|挂载点|分区类型|挂载说明|
|-------|--------|------|--------|
|/      |        |overlayfs||
|       |/rom    |squashfs|mtd:rootfs 即rootfs分区|
|       |/overlay|jffs2|/dev/mtdblock12即rootfs_data分区|

而其他挂载点基本上都是```tmpfs```，即内存文件系统。

### 系统特点
* Squashfs用来存储系统必须的文件；
* Squashfs具有更好的压缩率，能够最大限度的节约Flash空间；
* Squashfs为只读，能够在系统发生异常时，恢复到默认状态；
* Jffs2为可写，且也是一种压缩文件系统，同时能够保证磨损均衡
* 从使用的角度看，整个文件系统可写，且系统重启后仍然后能保存之前的修改；

### 升级对分区的影响
#### 恢复出厂设置
升级时，如果要求恢复出厂设置，那么会直接调用```jffs2reset -y```对rootfs_data区进行擦除动作；

#### 保存配置
* 由于rootfs_data是一个独立分区，并不需要做特别的处理。也因此之前对系统的修改也会被保存下载。    
* 再加上overlay文件系统的特点，会优先使用```/overlay```目录下的文件，即优先使用rootfs_data分区的内容，也就是说还是老的，被修改过的内容，而不是新升级的固件中的内容。

## 前世今生穿越大法
### 穿越准备工作
需要手动修改```qsdk/package/base-files/files/lib/preinit/80_mount_root```中的do_mount_root函数为以下内容：
```bash
do_mount_root() {
        echo "Before mount_root"
        magic_boot=`cat /proc/cmdline | grep 'magic_boot'`
        if [ -n "$magic_boot" ];then
                echo "do rootfs_data overlay"
                mount_root
        else
## use ram overlay, avoid write files
                do_ram_overlay
        fi
        boot_run_hook preinit_mount_root
        [ -f /sysupgrade.tgz ] && {
                echo "- config restore -"
                cd /
                tar xzf /sysupgrade.tgz
        }
        do_init_config
        echo "After mount_root"
}
```
或者直接下载bug20583中的patch文件change_rootfs_data_mount_point_v3.patch    
[直接下载PATCH](http://redmine.skspruce.net/redmine/attachments/download/38533/change_rootfs_data_mount_point_v3.patch)

### 穿越前世
```bash
fw_setenv fsbootargs 'root=mtd:rootfs rootfstype=squashfs rootwait magic_boot'
firstboot
```
### 回到今生
```bash
fw_setenv fsbootargs 
firstboot
```

### 穿越注意事项
***要穿越，必须做恢复出厂动作，否则保存的配置文件无法同步***

## WIA3200-80s的文件系统
### 分区表
要查看当前的分区表，最好最直接的方式就是dmesg -c查看设备启动时的输出比如：
```bash
[    1.068000] mtd .name = raspi, .size = 0x01000000 (0M) .erasesize = 0x00000010 (0K) .numeraseregions = 65536
[    1.068000] Creating 10 MTD partitions on "raspi":
[    1.068000] 0x000000000000-0x000001000000 : "ALL"
[    1.068000] 0x000000000000-0x000000030000 : "Bootloader"
[    1.072000] 0x000000030000-0x000000040000 : "Config"
[    1.072000] 0x000000040000-0x000000050000 : "Factory"
[    1.072000] 0x000000050000-0x000000ad0000 : "firmware"
[    1.076000] 0x000000187ddf-0x000000ad0000 : "rootfs"
[    1.076000] mtd: partition "rootfs" must either start or end on erase block boundary or be smaller than an erase block -- forcing read-only
[    1.076000] mtd: partition "rootfs_data" created automatically, ofs=0x910000, len=0x1c0000
[    1.076000] 0x000000910000-0x000000ad0000 : "rootfs_data"
[    1.076000] 0x000000ad0000-0x000000b50000 : "reserve"
[    1.080000] 0x000000b50000-0x000000fb0000 : "firmware2"
[    1.080000] 0x000000fb0000-0x000000fe0000 : "Bootloader2"
[    1.084000] 0x000000fe0000-0x000000ff0000 : "board"
[    1.084000] 0x000000ff0000-0x000001000000 : "Reboot Cause"
[    1.088000] rdm_major = 253
```
从上面的分区的起始和结束地址可以看出***firmware***分区被自动切割成了*内核* 和***rootfs***；    
而***rootfs***又被分割了一部分出来作为***rootfs_data***分区。

### 挂载情况
通过mount可以查看当前的系统目录挂载情况
```bash
root@OpenWrt:~# mount
rootfs on / type rootfs (rw)
/dev/root on /rom type squashfs (ro,relatime)
proc on /proc type proc (rw,noatime)
sysfs on /sys type sysfs (rw,noatime)
tmpfs on /tmp type tmpfs (rw,nosuid,nodev,noatime)
/dev/mtdblock6 on /overlay type jffs2 (rw,noatime)
overlayfs:/overlay on / type overlayfs (rw,noatime,lowerdir=/,upperdir=/overlay)
tmpfs on /dev type tmpfs (rw,relatime,size=512k,mode=755)
devpts on /dev/pts type devpts (rw,relatime,mode=600)
debugfs on /sys/kernel/debug type debugfs (rw,noatime)
/dev/mtdblock7 on /mnt/mtdblock7 type jffs2 (rw,relatime)
```
这个和之前的Taurus的情况类似，不再详述。
|-------|挂载点|分区类型|挂载说明|
|-------|--------|------|--------|
|/      |        |overlayfs||
|/dev/root|/rom  |squashfs|即rootfs分区|
|/dev/mtdblock6|/overlay|jffs2|即rootfs_data分区|

而其他挂载点基本上都是```tmpfs```，即内存文件系统。

### 系统特点
这个和之前的Taurus的情况类似，不再详述。

### 升级对分区的影响
#### 恢复出厂设置
升级时，如果要求恢复出厂设置，那么会直接调用```jffs2reset -y```对rootfs_data区进行擦除动作；

#### 保存配置
> 这个地方和之前的Taurus有重大的区别    

* 由于rootfs_data不是一个独立分区，是把***firmware***分区分割后再分割***rootfs***产生的；
* 在升级的时候，***rootfs***的大小是有可能发生变化的，也就是***rootfs_data***的分区大小是有可能在升级后发生变化，因此在升级之前需要先把文件备份起来；    
1. 打包需要保存的文件（根据一个文件列表）；
2. 写入固件；
3. 把打包好的文件附着在新升级的rootfs分区尾部；
4. 重新启动系统；
5. 切割出附着的打包文件并放入内存；
6. 格式化新切割出来的rootfs_data分区并挂载；
7. 解压已经放入内存的打包文件；
8. 搞定。

* 由于在升级之前只打包了指定列表的文件，也就是说没有被指定的文件是在升级后就会被清除。

## 参考链接
* [Filesystem](https://wiki.openwrt.org/doc/techref/filesystems)
