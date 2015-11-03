=========================================================
Buildroot 龙芯1C支持指南
=========================================================

引子：从龙芯1C预订拿到板子已经很长一段时间了，因为各种事情，一直让它呆在角落的冷宫里。昨天，愤而出去骑行，有导航出错，入的一片幽静山林，正能量爆棚，一下午休息，晚上兴致上来，看了看龙芯的说明，对rootfs部分的构建太过于麻烦，于是夜深人静，开始折腾，经由几个小时鼓捣，终于让buildroot可以支持龙芯1C-智龙开发板rootfs构建。高兴之余，觉得可以将过程写下来，可以让大家了解buildroot的构建机制，对编译工具链选择，系统包指定，以及最后的文件系统打包等都有一个细节的了解。

关于智龙开发板
=========================================================

智龙开发板是由 openloongson 开源社区推出的基于龙芯1C处理器的精简开发板，提供基础的硬件接口，包括一路以太网接口，一路USB Host接口，SD卡存储，2x40 pin io扩展，外置纽扣电源供电RTC。具体的信息可以参考开源龙芯社区网站和论坛。

龙芯 1C 介绍
----------------------------------------------------------

龙芯1C芯片是基于LS232处理器核的高性价比单芯片系统，具备丰富的外设接口及片上模块，为开发者提供足够的计算能力和多应用的连接能力。可应用于指纹生物识别、物联传感等领域。

龙芯1C包含浮点处理单元，可以有效增强系统浮点数据处理能力。1C的内存接口，支持多种类型的内存，允许灵活的系统设计。支持8-bit SLC NAND或MLC NAND FLASH，提供高容量的存储扩展接口。

龙芯1C的具体用户手册和数据手册可以查看 http://www.loongson.cn/product_info.php?id=53 。

要点准备
----------------------------------------------------------

1. 龙芯处理器兼容MIPS32指令集，而且官方提供消息，在新的gcc中是对龙芯各类芯片提供主流的支持。那么也就是说我们可以通过MIPS32的GCC编译工具链编译出能够支持龙芯1C的可执行文件。

2. 目前在开源社区存在两个非常广泛使用的Linux from scratch 开发平台，即 openWRT 和 Buildroot，两者都对MIPS32提供支持。这里我们选择Buildroot作为基础工具构建适用于龙芯1C的rootfs。

3. 根据龙芯开源社区或者网上有限的参考资料，智龙开发板使用yaffs2作为根分区文件系统，并且整个Nand Flash分为三个分区。第一个分区为内核分区，第二个分区为根分区，第三个分区为数据区。对应的分区表如下：
   
   .. code:: shell

      # cat /proc/mtd 
      dev:    size   erasesize  name
      mtd0: 00e00000 00020000 "kernel"
      mtd1: 06400000 00020000 "os"
      mtd2: 00e00000 00020000 "data"


4. 龙芯使用网络烧录 rootfs 指令如下：

   基于 *cramfs* 文件系统镜像烧录指令：
   
   .. code:: shell
   
      PMON>devcp tftp://192.168.x.xxx/rootfs-cramfs.img /dev/mtd1
   
   基于 *jffs2* 文件系统镜像烧录指令：
   
   .. code:: shell
       
      PMON>devcp tftp://192.168.x.xxx/rootfs-jffs2.img /dev/mtd1
      
   基于 *yaffs2* 文件系统镜像烧录指令：
   
   .. code:: shell
   
      PMON>mtd_erase /dev/mtd1
      PMON>devcp tftp://192.168.x.xxx/rootfs-yaffs2.img /dev/mtd1 yaf nw
      
5. 龙芯启动参数设置指令如下：

   基于 *cramfs* 文件系统的启动参数设置：
   
   .. code:: shell
   
      PMON>set append 'root=/dev/mtdblock1 console=ttyS2,115200 rootfstype=cramfs video=ls1bfb:480x272-16@70'
 
   基于 *jffs2* 文件系统的启动参数设置：
   
   .. code:: shell
   
      PMON>set append 'root=/dev/mtdblock1 console=ttyS2,115200 rootfstype=jffs2 video=ls1bfb:480x272-16@70'

   基于 *yaffs2* 文件系统的启动参数设置：
   
   .. code:: shell
   
      set append 'root=/dev/mtdblock1 console=ttyS2,115200 rootfstype=yaffs2 video=ls1bfb:480x272-16@70'

   PMON 中的系统重启指令：
   
   .. code:: shell
   
      PMON>reboot
      
6. 龙芯的根文件系统打包方法：

   基于 *cramfs* 文件系统打包：
  
   .. code:: shell
  
      mkcramfs /root/rootfs rootfs-cramfs.img
      chmod 777 rootfs-cramfs.img
     
      # 或者自带工具
      mkfs.cramfs /root/rootfs rootfs-cramfs.img
      chmod 777 rootfs-cramfs.img
     
   .. note::
  
      使用 `chmod 777 rootfs-cramfs.img` 修改文件系统权限，是为了防止出现无法烧写的情况。
     
   基于 *jffs2* 文件系统打包：
  
   .. code:: shell
  
      mkfs.jffs2 -r /root/rootfs -o rootfs-jffs2.img -e 0x20000 --pad=0x2000000 -n
      chmod 777 rootfs-jffs2.img
     
   基于 *yaffs2* 文件系统打包：
  
   .. code:: shell
  
      mkyaffs2image /root/rootfs rootfs-yaffs2.img
      chmod 777 rootfs-yaffs2.img
     
   .. note::
  
      这里需要注意的是，打包 yaffs2 文件系统镜像所使用的命令为 *mkyaffs2image* 而不是 Buildroot 中默认打包 yaffs2 的 *mkyaffs2* 指令。两者由不同的软件包生成，命令也不相同。
     
Buildroot MIPS 构建
=========================================================

在拿到智龙开发板，并了解了上面的准备工作，就可以开始 MIPS 版本的 Buildroot 构建。目前使用的 Buildroot 来自于 buildroot 的 `github 开发版本 <https://github.com/buildroot/buildroot>`_。构建更多详细信息可以参考它的 `操作手册 <http://buildroot.org/downloads/manual/manual.html>`_。

1. 获取 Buildroot 源码
    
   .. code:: shell
  
      git clone https://github.com/pengphei/buildroot.git
      
2. 安装准备软件

   buildroot 的构建需要依赖以下软件或者库：
   
   * which
   * sed
   * make (3.81及以上版本)
   * binutils 
   * build-essential (仅用于Debian衍生版本)
   * gcc (2.95及以上版本)
   * g++ (2.95及以上版本)
   * bash
   * patch
   * gzip
   * bzip2
   * perl (5.8.7及以上版本)
   * tar
   * cpio
   * python (2.6及以上版本)
   * unzip
   * rsync
   * wget
   * ncurses5 (用于 menuconfig 命令行配置界面)
   * qt4 (用于 xconfig 配置界面)
   * glib2, gtk2, glade2 (用于 gconfig 配置界面)
   * 其他的配置，需要根据构建过程中的问题，相应进行添加。

3. 系统构建

   为了构建出支持智龙开发板可用的镜像格式。在原本的 Buildroot 环境中添加了 yaffs2img 文件系统支持。该包与 Buildroot 中原有的 yaffs2 文件系统构建并不相同，需要特别注意。
   
   智龙开发板的构建命令如下：
   
   .. code:: shell
   
      cd buildroot
      cp configs/loongson1c_smartloong_defconfig .config
      make
   
   如果希望根据自己的需要，添加自己需要的软件包，可以执行如下命令进行根文件系统软件包的定制：
   
   .. code:: shell
   
      make menuconfig
      
   .. note::
   
      上述命令与 linux 内核的配置工具和使用方法完全相同。只是 buildroot 将会从网络上下载自己所需要的基础软件包以及构建工具链，在首次构建时，代码的下载和构建将会需要比较长的时间。
  
   在 *make* 命令执行完成之后，会生成 *output/images/rootfs.yaffs2img* 文件，该文件即为可以烧录到智龙开发板中的根文件系统镜像。
   
4. 镜像烧录

   在完成上述的镜像构建后，可以通过 *tftp* 将最终的根文件系统镜像烧录到智龙开发板，相应的指令如下：
   
   * 在开机后，马上按下空格键，进入 PMON 环境。
   * 在进入 *PMON* 环境后，执行如下指令序列完成镜像烧录，和重新启动系统。
   
     擦出 rootfs 分区以及烧录 rootfs 镜像：
     
     .. code:: shell
     
        PMON> mtd_erase /dev/mtd1
        PMON> devcp tftp://192.168.x.xxx/rootfs-yaffs2.img /dev/mtd1 yaf nw
        
     设置内核启动参数：
     
     .. code:: shell
   
        PMON> set append 'root=/dev/mtdblock1 console=ttyS2,115200 rootfstype=yaffs2'
        
     重启系统：
     
     .. code:: shell
     
        reboot
        

     
     
   
      
