---
layout: post
title: Android Camera
categories:
- programmer
tags:
- android
---



Android Camera 框架从整体上看是一个 client/service 的架构,	
有两个进程:	
client 进程,可以看成是 AP 端,主要包括 JAVA 代码与一些 native c/c++代码;	
service 进 程,属于服务端,是 native c/c++代码,主要负责和 linux kernel 中的 camera driver 交互,搜集 linuxkernel 中 cameradriver 传上来的数据,并交给显示系统显示。
client 进程与 service 进程通过 Binder 机制通信, client 端通过调用 service 端的接口实现各个具体的功能。






总体介绍 
Android Camera 框架从整体上看是一个 client/service 的架构,
有两个进程:
一个是 client 进程,可以看成是 AP 端,主要包括 JAVA 代码与一些 native c/c++代码;
另一个是 service 进程,属于服务端,是 native c/c++代码,主要负责和 linux kernel 中的 camera driver 交互,搜 
集 linux kernel 中 camera driver 传上来的数据,并交给显示系统(surface)显示。
client 进程与service 进程通过 Binder 机制通信
client 端通过调用 service 端的接口实现各个具体的功能。 
需要注意的是真正的 preview 数据不会通过 Binder IPC 机制从 service 端复制到 client 端, 
但会通过回调函数与消息的机制将 preview 数据 buffer 的地址传到 client 端, 最终可在 JAVA AP 
中操作处理这个 preview 数据




Preview 数据流程 
Android 框架中 preview 数据的显示过程如下: 
1、 打开内核设备文件。CameraHardwareInterface.h 中定义的 openCameraHardware()打开 
      linux kernel 中的 camera driver 的设备文件(如/dev/video0)      ,创建初始化一些相关的类 
      的实例。 
2、 设置摄像头的工作参数。CameraHardwareInterface.h 中定义的 setParameters()函数,在 
      这一步可以通过参数告诉 camera HAL 使用哪一个硬件摄像头,  以及它工作的参数  (size, 
    format 等) ,并在 HAL 层分配存储 preview 数据的 buffers(如果 buffers 是在 linux kernel 
    中的 camera driver 中分配的,在这一步也会拿到这些 buffers mmap 后的地址指针)                      。 
3、 设置显示目标。需在 JAVA APP 中创建一个 surface 然后传递到 CameraService 中。会调 
    用到 libcameraservice.so 中的 setPreviewDisplay(const sp & surface)函数中。在 
    这里分两种情况考虑:一种是不使用 overlay;一种是使用 overlay 显示。如果不使用 
    overlay 那设置显示目标最后就在 libcameraservice.so 中,不会进 Camera HAL 动态库。 
    并将上一步拿到的 preview 数据 buffers 地址注册到 surface 中。 如果使用 overlay 那在 
    libcameraservice.so 中会通过传进来的 Isurface 创建 Overlay 类的实例,然后调用 
    CameraHardwareInterface.h 中定义的 setOverlay()设置到 Camera HAL 动态库中。 
4、 开始 preview 工作。最终调用到 CameraHardwareInterface.h 中定义的 startPreview()函数。 
    如果不使用 overlay,Camera HAL 得到 linux kernel 中的 preview 数据后回调通知到 
    libcameraservice.so 中。在 libcameraservice.so 中会使用上一步的 surface 进行显示。如 
    果使用 overlay,    Camera HAL 得到 linux kernel 中的 preview 数据后直接交给 Overlay 对象, 
    然后有 Overlay HAL 去显示。





V4L2
V4L（video4linux是一些视频系统，视频软件、音频软件的基础，经常时候在需要采集图像的场合，如视频监控，webcam,可视电话，经常使用在embedded linux中是linux嵌入式开发中经常使用的系统接口。
它是linux内核提供给用户空间的编程接口，各种的视频和音频设备开发相应的驱动程序后，就可以通过v4l提供的系统API来控制视频和音频设备，
也就是说v4l分为两层，
底层为音视频设备在内核中的驱动，
上层为系统提供的API，而对于我们来说需要的就是使用这些系统API。

V4L2较V4L1有较大的改动，并已成为2.6的标准接口。下边先就V4L2在视频捕捉或camera方面的应用框架。
V4L2采用流水线的方式，操作更简单直观，基本遵循打开视频设备、设置格式、处理数据、关闭设备，更多的具体操作通过ioctl函数来实现。








Android Camera 调用流程总结 

1.总体介绍
  Android Camera框架从整体上看是一个client/service架构。有两个进程，一个是client进程，可以看成AP端
，主要包括Java代码和一些native层的c/c++代码；另一个是service进程，属于服务端，是native c/c++代码，
主要负责和linux kernel中的camera driver交互，搜集linux kernel中driver层传上来的数据，并交给显示系统（surface）显示。client 和 service 进程通过Binder机制进行通信，client端通过调用service端的接口实现各个具体的功能。
  对于preview数据不会通过Binder机制从service端copy 到client端，但会通过回调函数与消息机制将preview数据的buffer地址传到client端，最终可在Java ap中操作处理preview数据。
2.调用层次划分
Package -> Framework -> JNI ->Camera.cpp -- (binder) ->CameraService ->Camera HAL -> Qcom ->Camera Driver
client端：
Package 中的 camera.java 调用Framework中的 camera.java（framework/base/core/java/android/hardware).
Framework中的camera.java 调用 JNI层的native 函数。JNI层的调用实现在android_hardware_camera.cpp（framework/base/core/jni文件下的文件都被编译进libandroid_runtime.so）文件中，android_hardware_camera.cpp文件中的register_android_hardware_camera(JNIEnv *env)函数会将native函数注册到虚拟机中，以供framework层的JAVA代码调用，这些native函数通过调用libcamera_client.so中的camera类实现具体功能。
  核心的libcamera_client.so动态库源代码位于：framework/base/core/av中，其中Icamera,IcameraClient,IcameraService三个类按照Binder IPC通信要求的框架实现的，用来与service端通信。CameraParameters类接受framework层的android.hardware.camera::Parameters类为参数。
service端:
service端的实现在动态库libcameraservice.so中，源代码位于:frameworks/av/services/camera。
CameraService:Client类通过调用Camera HAL层来实现具体的功能。
Camera Service 在系统启动时new了一个实例额，以“media.camera”注册到servicemanager中。在init.rc中启动多媒体服务进程。

CameraHAL层：
libcameraservice.so::CameraService::Client类调用camera HAL 的代码实现具体功能。
JAVA Ap中的功能调用最终会调用到HAL层，HAL层通过startpreview 掉到hardware/qcom/camera中的start_preview.然后就是高通这一层对底层驱动上来的数据做一些处理。从linux kernel中的camera driver得到preview数据。然后交个surface显示或者保存到文件。









##参考资料
--------------------------------------------------------------
1	Android Camera 运行流程	
	http://blog.csdn.net/unicornkylin/article/details/13293295	
2	介绍 Android 的 Camera 框架	
	http://www.open-open.com/lib/view/open1328063735233.html	
3	Android Camera架构浅析	
	http://blog.csdn.net/BonderWu/article/details/5814278	
4	Android实时视频采集—Camear	
	http://blog.csdn.net/smart_yujin/article/details/10515431	
5	Android Camera 通过V4L2与kernel driver的完整交互过程	
	http://blog.chinaunix.net/uid-26215986-id-3552456.html	
6	V4L2视频应用程序编程架构	
	http://blog.chinaunix.net/uid-26215986-id-3552455.html	
7	Camera--V4L2驱动学习记录	
	http://blog.chinaunix.net/uid-26215986-id-3552453.html	
8	虚拟视频驱动程序vivi.c源码分析	
	http://blog.chinaunix.net/uid-26215986-id-3552454.html	
9	V4L2用户空间和kernel层driver的交互过程	
	http://blog.chinaunix.net/uid-26215986-id-3552458.html	
10	FS_S5PC100平台上Linux Camera驱动开发详解（一）	
	http://blog.chinaunix.net/uid-26215986-id-3059129.html	
12	Android Camera数据流分析全程记录（overlay方式一）	
	http://blog.chinaunix.net/uid-26215986-id-3573400.html	
13	Android Camera之SurfaceView学习	
	http://blog.chinaunix.net/uid-26215986-id-4033440.html	



--------------------------------------------------------------


Android display架构分析
http://blog.csdn.net/bonderwu/article/details/5805961

	
Android Camera CameraHal.cpp 初始化分析	
http://blog.chinaunix.net/uid-26215986-id-4033445.html	

















	
























