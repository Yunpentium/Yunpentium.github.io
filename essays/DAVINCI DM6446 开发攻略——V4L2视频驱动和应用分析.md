[转载] http://zjbintsystem.blog.51cto.com/964211/464729/

2010年即将过去，有很多感慨需要总结一下，自从2010年1月份开始着手写DM6446开发攻略以来，得到很多网友和客户对本人博客的支持，同时结识一些有诚意的客户，他们对本人和我们团队的认可，在这里表示感谢。特别是南京航空航天大学的一个非常有个性、有良知和责任的李博导，对本团队的项目设计速度的赞赏，让本人非常感动。潜水2个多月没有更新博客，多少有点对不住51CTO的关照，在另一款新产品出来前，DM6446开发攻略的2010年最后一篇文章，本人决定从做DAVINCI产品的角度，认真写好最有价值的视频处理。DAVINCI产品的视频驱动和应用，是非常重要的因素，就像LINUX的网络驱动和应用一样，没有网络，还叫LINUX吗？DAVINCI没有视频处理，还叫DAVINCI吗？

针对DAVINCI DM6446平台，网络上也有很多网友写了V4L2的驱动，但只是解析Montavista linux-2.6.10 V4L2的原理、结构和函数，深度不够。本文决定把Montavista 的Linux-2.6.18 V4L2好好分析一下，顺便讲解在产品中的应用，满足一些客户提出要求，毕竟V4L2是LINUX一个很重要的视频驱动，适合很多嵌入式芯片平台。本文首先讲解DM6446 DAVINCI视频处理技术的硬件工作原理，然后讲解DM6446 V4L2采集驱动和输出驱动，然后对TI DVSDK2.0里边提供的V4L2的例子进行详细讲解，怎样和驱动配合起来。
 
### 第一节 DAVINCI视频处理硬件   

有关DM6446 DAVINCI视频处理技术，有两个文档：VPFE sprue38ec.pdf和VPBE sprue37c.pdf，必须要看看的。下图是DAVINCI视频处理技术的框图，VPSS（视频处理子系统）包含VPFE和VPBE，VPFE负责前端视频采集和处理，而VPBE负责后端视频输出，通过OSD和VECN直接输出到DACS（数字转模拟输出口，一共4个通道DAC，通过外围视频编码芯片转换成复合视频CVBS输出到普通电视机）或者直接输出到LCD（DM6446支持RGB24位信号输出到数字LCD屏，4.3寸，7寸屏等）。

![这里写图片描述](http://img.blog.csdn.net/20170625210504446?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMDE4NjAwMQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
图1 DAVINCI VPSS框图
 
从图1可以看出，VPFE系统可以接CCD或者CMOS sensor，同时也可以接视频解码芯片，目前Montavista Linux-2.6.18驱动给出的TI EVM驱动支持MT9T001 CMOS芯片和TVP5146视频解码芯片，VPFE采用RAW模式控制MT9T001 CMOS芯片，数码相机产品基本是这种应用方式，而VPFE采用BT601或BT656的方式控制TVP5146视频解码芯片，很多做安防、机器视觉等的方案都是这种模式，因为这种方式最普通，视频前端买个普通的CCD摄像机，接条视频线和电源，就可以用通过类似TVP5146的芯片采集到图像了，本人也着重介绍这种情况。
而图-1里边的Resizer（图像缩放1/4x~4x）、Preview（预览器）、H3A（硬件自动白平衡、自动对焦、自动曝光）、Histogram（直方图）是对采集到的视频进行处理，一般常用到的是Resizer，不需要占用ARM和DSP的资源，对采集到的YUV422数据进行处理，然后才提交给H264等算法进行压缩，这一点可以在dvsdk_2_00_00_22\dvsdk_demos_2_00_00_07\dm6446里边的例子体现到。

VPBE系统可以对处理后的视频（VIDEO）数据或图像(IMAGE)进行处理和输出，一般用户可以通过OSD功能叠加自己的LOGO、字符、时间、坐标、框图等信息，然后通过VENC模块输出到DAC或者LCD接口。

VPFE和VPBE所有的数据交换都是在DDR上处理，VPFE采集的视频数据，比如YUV422格式（U0Y0V0Y1）都有指定的DDR地址，而VPBE也有另外指定的DDR地址。
 
### **第二节   V4L2采集驱动**
 
对应上面的硬件处理过程，软件工程师最关心的是如何配置VPFE和VPBE的寄存器，如何实现DDR的视频数据视频缓冲处理，在LINUX内核里如何实现DMA处理。Montavista 的Linux-2.6.18 V4L2驱动源码已经帮客户实现VPFE和VPBE的处理，他们的源码目录是linux-2.6.18_pro500\drivers\media\video\和linux-2.6.18_pro500\drivers\media\video\davinci目录。对于LINUX驱动工程师，首先先按以下三个图配置Montavista linux-2.6.18_pro500的内核，让linux-2.6.18_pro500支持V4L2。
 ![这里写图片描述](http://img.blog.csdn.net/20170625210529047?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMDE4NjAwMQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
图2 配置Multimedia devices

按图2选择Video For Linux，然后进入“Video Capture Adapters”，按图-3配置DAVINCI视频采集选项， 
![这里写图片描述](http://img.blog.csdn.net/20170625210603036?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMDE4NjAwMQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
图 3 配置采集选项

 同在一个配置界面，选择和进入“Encoders and Decoders”，配置VPBE实现视频输出处理。
 ![这里写图片描述](http://img.blog.csdn.net/20170625210615109?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMDE4NjAwMQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
图-4 配置VPBE DISPLAY选项
 
然后再去到驱动driver文件夹找到相关文件。
**1.  VPFE采集涉及到下面几个文件：**   
- ccdc_davinci.c（主要工作把重点放在ccdc_config_ycbcr（）函数上，主要是配置是否采用BT656或BT601，采集视频的数据格式等）   
 -  tvp5146.c（这个就是你板子外围视频采集芯片驱动，一般是I2C访问，有些朋友的项目可以是TVP5150，我们开发板的是TVP5158，还可以是Techwell的芯片和nextchip的芯片参考TVP5146的驱动，把自己的项目芯片驱动移植过来就OK了）
 - davinci_vpfe.c（这就是和应用程序打交道的内核程序，和v4l2配合使用，供应用层调用）
**2. VPBE输出涉及到： **   
davinci_display.c
davinci_enc_mngr.c
davinci_osd.c
davinci_platform.c
vpbe_encoder.c
**3. V4L2驱动：**
在linux-2.6.18_pro500\drivers\media\video目录里，
v4l2-common.c
video-buf.c
videodev.c
 
这些函数比较复杂，V4L2必须和davinci_vpfe.c、vpbe_encoder.c配合，借助davinci_vpfe.c提供的接口函数，LINUX应用程序才能控制TVP5156和CCDC采集；V4L2负责对采集到的视频数据进行管理，包括DMA和缓冲处理，Montavista已经帮客户搞定这些，我们需要的做的就是学习人家是如何实现这些处理的。

本公司的开发板采用的是TVP5158芯片，见http://zjbintsystem.blog.51cto.com/964211/392221。移植修改基本上是上面的几个VPFE采集源文件。至于我们板子上的美光MT9M112、MT9D112等类似CMOS芯片的控制，基本也是修改上面的几个VPFE采集源文件。
 
### **第三节   V4L2例子源码分析**
 
在dvsdk_2_00_00_22\PSP_02_00_00_140\examples\dm644x\v4l2里，有V4L2应用的例子，里边有v4l2_mmap_loopback.c和v4l2_userptr_loopback.c，我们主要分析v4l2_mmap_loopback.c。
很多网友介绍LINUX V4L2视频原理都是从本节开始的，以TVP5146采集芯片为例。   

#### **1、Makefile修改**
下面的Makefile的内容也适合于其他LINUX应用程序，

	# Makefile for v4l2 application
	CROSSCOMPILE = arm_v5t_le-
	CC=$(CROSSCOMPILE)gcc
	LD=$(CROSSCOMPILE)ld
	OBJCOPY=$(CROSSCOMPILE)objcopy
	OBJDUMP=$(CROSSCOMPILE)objdump
	INCLUDE = /home/davinci/dm6446/ty-dm6446-1000/linux-2.6.18_pro500/include
	（本人把产品级的linux-2.6.18_pro500放到上面的目录，ty-dm6446-1000是本人公司深圳桐烨科技的一个DM6446产品）
	all: tvp5146_v4l2_mmap
	 
	tvp5146_v4l2_mmap: v4l2_mmap_loopback.c
	      $(CROSSCOMPILE)gcc -Wall -O2 v4l2_mmap_loopback.c -I $(INCLUDE) -o tvp5146_v4l2_mmap
	      $(CROSSCOMPILE)strip tvp5146_v4l2_mmap
	      cp -f tvp5146_v4l2_mmap/home/davinci/nfs/tirootfs/opt/app/
	（自动COPY到NFS进行调试）
	 
	%.o:%.c
	      $(CC) $(CFLAGS) -c $^
	 
	clean:
	      rm -f *.o *~ core tvp5146_v4l2_mmap
   
#### **2、下面通过分析v4l2_mmap_loopback.c的源码，从应用层的角度讨论V4L2的原理**
 
	#include <stdio.h>
	#include <fcntl.h>
	#include <string.h>
	#include <getopt.h>
	#include <stdlib.h>
	。。。。。。。。。。。。。。。。。。。。。。。。。。
	#include <fcntl.h>
	#include <time.h>
	/*以上指向你安装的linux主机/usr/include*/
	#include <linux/fb.h>/*指向Montavista linux-2.6.18\include\linux*/
	#include <asm/types.h>/*指向linux-2.6.18\include\asm-arm*/
	 
	/* Kernel header file, prefix path comes from makefile */
	#include <media/davinci/davinci_vpfe.h>/*指向linux-2.6.18 \include\media\davinci*/
	#include <video/davincifb_ioctl.h>/*指向linux-2.6.18 \include\video*/
	#include <linux/videodev.h>
	#include <linux/videodev2.h>
	#include <media/davinci/davinci_display.h>
	#include <media/davinci/ccdc_davinci.h>
	 
	/*   LOCAL DEFINES*/
	#define CAPTURE_DEVICE      "/dev/video0" 文件系统中采集驱动用到的设备节点
	 
	#define WIDTH_NTSC         720
	#define HEIGHT_NTSC              480   视频NTSC制式
	#define WIDTH_PAL            720
	#define HEIGHT_PAL          576   视频PAL支持
	 
	#define MIN_BUFFERS       2    采集时存放YUV视频数据的缓冲数，做到乒乓BUFFER，
	 
	#define UYVY_BLACK       0x10801080无图像的YUV值处理
	 
	/* Device parameters */
	#define VID0_DEVICE "/dev/video2"
	文件系统中DISPLAY输出设备节点（对照内核驱动davinci_display.c）
	#define VID1_DEVICE "/dev/video3"文件系统中DISPLAY输出设备节点
	#define OSD0_DEVICE       "/dev/fb/0"文件系统中OSD0设备节点
	#define OSD1_DEVICE       "/dev/fb/2"文件系统中OSD1设备节点
	 
	/* Function error codes */
	#define SUCCESS          0
	#define FAILURE           -1
	 
	/* Bits per pixel for video window */
	#define YUV_422_BPP 16
	#define BITMAP_BPP_8      8
	 
	#define DISPLAY_INTERFACE       "COMPOSITE"显示输出定义复合视频输出
	#define DISPLAY_MODE_PAL        "PAL"显示输出定义PAL制输出
	#define DISPLAY_MODE_NTSC            "NTSC"显示输出定义NTSC制输出
	 
	#define round_32(width)       ((((width) + 31) / 32) * 32 ) 字节对齐
	 
	/* Standards and output information */
	#define ATTRIB_MODE              "mode"
	#define ATTRIB_OUTPUT          "output"
	 
	#define LOOP_COUNT        500 本例子采集多少帧就停止运行（PAL制每秒25帧）
#### **3、V4L2的操作流程**
**Video4linux2（简称V4L2)**，是linux中关于视频设备的内核驱动。在Linux中，视频设备是设备文件，可以像访问普通文件一样对其进行读写，摄像头在/dev/video0下。

**Video4linux2一般操作流程（视频设备）：**
1. 打开设备文件。 int fd=open(”/dev/video0″,O_RDWR);
2. 取得设备的capability，看看设备具有什么功能，比如是否具有视频输入等。VIDIOC_QUERYCAP,struct v4l2_capability
3. 选择视频输入，一个视频设备可以有多个视频输入。VIDIOC_S_INPUT,struct v4l2_input
4. 设置视频的制式和帧格式，制式包括PAL，NTSC，帧的格式个包括宽度和高度等。
VIDIOC_S_STD,VIDIOC_S_FMT,struct v4l2_std_id,struct v4l2_format
5. 向驱动申请帧缓冲，一般不超过5个。struct v4l2_requestbuffers
6. 将申请到的帧缓冲映射到用户空间，这样就可以直接操作采集到的帧了，而不必去复制。
7. 将申请到的帧缓冲全部入队列，以便存放采集到的数据.VIDIOC_QBUF,struct v4l2_buffer
8. 开始视频的采集。VIDIOC_STREAMON
9. 出队列以取得已采集数据的帧缓冲，取得原始采集数据。VIDIOC_DQBUF
10. 将缓冲重新入队列尾,这样可以循环采集。VIDIOC_QBUF
11. 停止视频的采集。VIDIOC_STREAMOFF
12. 关闭视频设备。close(fd);

**常用的结构体**(参见linux-2.6.18_pro500/include/linux/include/linux/videodev2.h)：

	struct v4l2_requestbuffers reqbufs;//向驱动申请帧缓冲的请求，里面包含申请的个数
	struct v4l2_capability cap;//这个设备的功能，比如是否是视频输入设备
	struct v4l2_input input; //视频输入
	struct v4l2_standard std;//视频的制式，比如PAL，NTSC
	struct v4l2_format fmt;//帧的格式，比如宽度，高度等
	struct v4l2_buffer buf;//代表驱动中的一帧
	v4l2_std_id stdid;//视频制式，例如：V4L2_STD_PAL
	struct v4l2_queryctrl query;//查询的控制
	struct v4l2_control control;//具体控制的值
 
从main()函数调用vpbe_UE_1（），在vpbe_UE_1（）里，可以看到采集流程和显示输出流程。
#### **4、V4L2采集过程**
initialize_capture（）里初始化采集配置、分配采集内存缓冲、启动开始采集。顺序调用init_capture_device（）+set_data_format（），init_capture_buffers（），start_streaming（）。
   
**1)、打开视频设备**
 在V4L2中，视频设备被看做一个文件。使用open函数打开这个设备：
/用非阻塞模式打开采集设备，见init_capture_device（）函数，fdCapture在本例子中定义为全局变量，

	       if ((fdCapture = open(CAPTURE_DEVICE, O_RDWR | O_NONBLOCK, 0)) <= -1) {
	              printf("InitDevice:open::\n");
	              return -1;
	       }
如果用阻塞模式打开采集设备，上述代码变为：

	if ((fdCapture = open(CAPTURE_DEVICE, O_RDWR, 0)) <= -1) {
	              printf("InitDevice:open::\n");
	              return -1;
	       }
 
关于阻塞模式和非阻塞模式，应用程序能够使用阻塞模式或非阻塞模式打开视频设备，如果使用非阻塞模式调用视频设备，即使尚未捕获到信息，驱动依旧会把缓存（DQBUFF）里的东西返回给应用程序。
 
**2）、设定属性及采集方式**
 打开视频设备后，可以设置该视频设备的属性，例如裁剪、缩放等。这一步是可选的。在Linux编程中，一般使用ioctl函数来对设备的I/O通道进行管理：

	extern int ioctl (int __fd, unsigned long int __request, …) __THROW;
	__fd：设备的ID，例如刚才用open函数打开视频通道后返回的fdCapture；
	__request：具体的命令标志符。
在进行V4L2开发中，一般会用到以下的命令标志符：

	1.    VIDIOC_REQBUFS：分配内存
	2.    VIDIOC_QUERYBUF：把VIDIOC_REQBUFS中分配的数据缓存转换成物理地址
	3.    VIDIOC_QUERYCAP：查询驱动功能
	4.    VIDIOC_ENUM_FMT：获取当前驱动支持的视频格式
	5.    VIDIOC_S_FMT：设置当前驱动的频捕获格式
	6.    VIDIOC_G_FMT：读取当前驱动的频捕获格式
	7.    VIDIOC_TRY_FMT：验证当前驱动的显示格式
	8.    VIDIOC_CROPCAP：查询驱动的修剪能力
	9.    VIDIOC_S_CROP：设置视频信号的边框
	10. VIDIOC_G_CROP：读取视频信号的边框
	11. VIDIOC_QBUF：把数据从缓存中读取出来
	12. VIDIOC_DQBUF：把数据放回缓存队列
	13. VIDIOC_STREAMON：开始视频显示函数
	14. VIDIOC_STREAMOFF：结束视频显示函数
	15. VIDIOC_QUERYSTD：检查当前视频设备支持的标准，例如PAL或NTSC。
这些IO调用，有些是必须的，有些是可选择的。他们可以从在内核中davinci_vpfe.c 里static int vpfe_doioctl(struct inode *inode, struct file *file,unsigned int cmd, void *arg)函数找到对应关系。
 
**3）、检查当前视频设备支持的标准和设置视频捕获格式**
 在set_data_format()函数里，检测完视频设备支持的标准后，还需要设定视频捕获格式：PAL制还是NTSC制，采集像素格式UYVY，奇偶场交错方式INTERLACED。
 
**4）、分配内存**
 接下来可以为视频捕获分配内存：
在init_capture_buffers（）里，使用VIDIOC_REQBUFS，我们获取了req.count个缓存，下一步通过调用VIDIOC_QUERYBUF命令来获取这些缓存的地址，然后使用mmap函数转换成应用程序中的绝对地址，最后把这段缓存放入缓存队列：

	// 读取缓存
	if (ioctl(fdCapture, VIDIOC_QUERYBUF, &buf) == -1) {
	return -1;
	}
	buffers[numBufs].length = buf.length;
	// 转换成相对地址
	buffers[nIndex].length = buf.length;
	buffers[nIndex].start =
	mmap(NULL, buf.length, PROT_READ | PROT_WRITE,
	MAP_SHARED, fdCapture, buf.m.offset);

**5）、启动开始采集**
 
	// 放入缓存队列
	if (ioctl(fdCapture, VIDIOC_QBUF, &buf) == -1) {
	return -1;
	}}
	/* all done , get set go */
	type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
	if (-1 == ioctl(fdCapture, VIDIOC_STREAMON, &type))
	printf("start_streaming:ioctl:VIDIOC_STREAMON:\n");
 
关于视频采集方式
 
操作系统一般把系统使用的内存划分成用户空间和内核空间，分别由应用程序管理和操作系统管理。应用程序可以直接访问内存的地址，而内核空间存放的是 供内核访问的代码和数据，用户不能直接访问。v4l2捕获的数据，最初是存放在内核空间的，这意味着用户不能直接访问该段内存，必须通过某些手段来转换地址。
一共有三种视频采集方式：使用read、write方式；内存映射方式和用户指针模式。
read、write方式:在用户空间和内核空间不断拷贝数据，占用了大量用户内存空间，效率不高。
内存映射方式：把设备里的内存映射到应用程序中的内存控件，直接处理设备内存，这是一种有效的方式。上面的mmap函数就是使用这种方式。
用户指针模式：内存片段由应用程序自己分配。这点需要在v4l2_requestbuffers里将memory字段设置成V4L2_MEMORY_USERPTR。
 
**6）、处理采集数据**
 V4L2有一个数据缓存，存放req.count数量的缓存数据。数据缓存采用FIFO的方式，当应用程序调用缓存数据时，缓存队列将最先采集到的 视频数据缓存送出，并重新采集一张视频数据。这个过程需要用到两个ioctl命令,VIDIOC_DQBUF和VIDIOC_QBUF：
 
	struct v4l2_buffer buf;
	memset(&buf,0,sizeof(buf));
	buf.type=V4L2_BUF_TYPE_VIDEO_CAPTURE;
	buf.memory=V4L2_MEMORY_MMAP;
	buf.index=0;
	//读取缓存
	if (ioctl(fdCapture, VIDIOC_DQBUF, &buf) == -1)
	{
	return -1;
	}
	//…………视频处理算法
	//重新放入缓存队列
	if (ioctl(fdCapture, VIDIOC_QBUF, &buf) == -1) {
	return -1;
	}
	关闭视频设备
	使用close函数关闭一个视频设备
	close(fdCapture)
 
#### **4、V4L2显示输出**
 
配置视频显示输出函数init_vid1_device（），初始化和采集差不多，这里就不用多解析，这个显示输出的例子通过DAC口，把采集的图像通过LOOPBACK方式，直接输出到普通电视机或DVD等视频IN的端口里，当然你的板子要有把DM6446 DAC信号通过视频放大器才能接到电视机上。从start_loop()函数里，下面的代码

      buf.type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
      buf.memory = V4L2_MEMORY_MMAP;

      /* determine ready buffer */
      if (-1 == ioctl(fdCapture, VIDIOC_DQBUF, &buf)) {
             if (EAGAIN == errno)
                    continue;
             printf("StartCameraCaputre:ioctl:VIDIOC_DQBUF\n");
             return -1;
      }

      /******************* V4L2 display ********************/
      displaybuffer = get_display_buffer(fd_vid1);
      if (NULL == displaybuffer) {
             printf("Error in getting the display buffer:VID1\n");
             return ret;
      }

      src = buffers[buf.index].start;
      dest = displaybuffer;

      /* Display image onto requested video window */
      for(i=0 ; i < dispheight; i++) {
             memcpy(dest, src, disppitch);
             src += disppitch;
             dest += disppitch;
      }
可以看出LOOPBACK方式的操作memcpy(dest, src, disppitch)，直接把采集的数据(720x576x2)字节放到视频输出缓冲dest，disppitch=1440,，就是一行UYVY的自己是1440。
 
### **第四节   DVSDK2.0有关V4L2的例子分析**
 
有上面的介绍，我们可以深入学习DM6446 DVSDK2.0有关V4L2的例子。

dvsdk_2_00_00_22\dvsdk_demos_2_00_00_07\dm6446里有encode，decode，encodedecode的例子，这些例子全部是应用程序。带有H264、MPEG4、G711这些算法的应用。


V4L2的例子函数为capture.c和display.c，他们不像第三节介绍的v4l2_mmap_loopback.c直接跟内核davinci_vpfe.c接口函数打交道，而是通过dmai，即dvsdk_2_00_00_22\dmai_1_20_00_06\packages\ti\sdo\dmai\linux目录下的源代码，跟内核davinci_vpfe.c、vpbe_encoder.c打交道，Montavista把内核驱动和VISA调用封装在一起。

dvsdk_2_00_00_22\dmai_1_20_00_06\packages\ti\sdo\_dmai\linux里的c文件就是V4L2和内核对接的源文件，好好学习这些例子，对大家做DAVINCI嵌入式产品非常有好处，本人也是从这些例子里学到很多LINUX的东西。
 
### **第五节 后记**
新的产品即将出来，根据一些客户的要求，我们重新对DM365/DM368进行第2轮PCB设计，我想很快就可以和大家探讨高清的方案，深圳市桐烨科技有限公司专门提供对应硬件平台和产品方案支持，我们专注ARM+DSP的产品方案和项目设计，IVS（智能视频监控）设计，810MHz的DM6446核心板将更加满足算法的要求。同时我们根据客户的项目需求，以深圳的速度（产品一条龙服务）帮客户设计产品。
