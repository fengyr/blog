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




##参考资料
--------------------------------------------------------------
1	Android Camera 运行流程	
	http://blog.csdn.net/unicornkylin/article/details/13293295	
2	介绍 Android 的 Camera 框架	
	http://www.open-open.com/lib/view/open1328063735233.html	
3	Android Camera架构浅析	
	http://blog.csdn.net/BonderWu/article/details/5814278	



--------------------------------------------------------------


Android display架构分析
http://blog.csdn.net/bonderwu/article/details/5805961


bing	
Android Camear 驱动

Android Camera CameraHal.cpp 初始化分析	
http://blog.chinaunix.net/uid-26215986-id-4033445.html	

Android Camera之SurfaceView学习
http://blog.chinaunix.net/uid-26215986-id-4033440.html

Android Camera数据流分析全程记录（overlay方式一）
http://blog.chinaunix.net/uid-26215986-id-3573400.html

Camera--V4L2驱动学习记录
http://blog.chinaunix.net/uid-26215986-id-3552453.html

Android Camera 调用流程总结
http://blog.chinaunix.net/uid-26215986-id-3522684.html

FS_S5PC100平台上Linux Camera驱动开发详解（一）
http://blog.chinaunix.net/uid-26215986-id-3059129.html

V4L2用户空间和kernel层driver的交互过程
http://blog.chinaunix.net/uid-26215986-id-3552458.html

Camera--V4L2驱动学习记录 
http://blog.chinaunix.net/uid-26215986-id-3552453.html

V4L2视频应用程序编程架构	
http://blog.chinaunix.net/uid-26215986-id-3552455.html	

Android Camera 通过V4L2与kernel driver的完整交互过程	
http://blog.chinaunix.net/uid-26215986-id-3552456.html	

--------------------------------------------------------------





Android&nbsp;实时视频采集—Camear…
http://blog.csdn.net/smart_yujin/article/details/10515431


















