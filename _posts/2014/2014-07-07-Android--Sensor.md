---
layout: post
title: Android Sensor
categories:
- programmer
tags:
- android
---



## Sensor 简要介绍
Sensor是很常见的重要器件，比如在生活常见到的声控开关，电子温度计等。手机与平板领域已经广泛使用sensor，
大家在使用这些设备的时候也许没有意识到sensor的存在，但确实已经和sensor打交道了。Android 1.5开始为开发人员
提供了关于sensor的API，这样开发人员可以开发出更加优秀的应用程序



## Android Sensors 种类

	ID	Sensor Tyoe								Desc
	-----------------------------------------------------------------------------------------------
	1	SENSOR_TYPE_ACCELEROMETER				加速度传感器又叫G-sensor，返回x、y、z三轴的加速度数值。
		加速度传感器								该数值包含地心引力的影响，单位是m/s^2
	-----------------------------------------------------------------------------------------------
	2	SENSOR_TYPE_MAGNETIC_FIELD				磁力传感器简称为M-sensor，返回x、y、z三轴的环境磁场数据
		磁力传感器								该数值的单位是微特斯拉（micro-Tesla），用uT表示。  
												硬件上一般没有独立的磁力传感器，磁力数据由电子罗盘传感器
												提供（E-compass）。
	-----------------------------------------------------------------------------------------------
	3	SENSOR_TYPE_ORIENTATION					方向传感器简称为O-sensor，返回三轴的角度数据，方向数据
		方向传感器								的单位是角度。 为了得到精确的角度数据，E-compass需要获
												取G-sensor的数据，  经过计算生产O-sensor数据，否则只能
												获取水平方向的角度。  方向传感器提供三个数据，分别为
												azimuth、pitch和roll。
	-----------------------------------------------------------------------------------------------
	4	SENSOR_TYPE_GYROSCOPE					陀螺仪传感器叫做Gyro-sensor，返回x、y、z三轴的角速度
		陀螺仪									数据。  角速度的单位是radians/second
	-----------------------------------------------------------------------------------------------
	5	SENSOR_TYPE_LIGHT						光线感应传感器检测实时的光线强度，光强单位是lux，其物理
		光线传感器								意义是照射到单位面积上的光通量。  光线感应传感器主要用于
												Android系统的LCD自动亮度功能。  可以根据采样到的光强数
												值实时调整LCD的亮度
	-----------------------------------------------------------------------------------------------
	6	SENSOR_TYPE_PRESSURE					压力传感器返回当前的压强，单位是百帕斯卡hectopascal（hPa）
		压力传感器
	-----------------------------------------------------------------------------------------------
	7	SENSOR_TYPE_TEMPERATURE					温度传感器返回当前的温度
		温度传感器
	-----------------------------------------------------------------------------------------------
	8	SENSOR_TYPE_PROXIMITY					接近传感器检测物体与手机的距离，单位是厘米。
		近场传感器								接近传感器可用于接听电话时自动关闭LCD屏幕以节省电量
	-----------------------------------------------------------------------------------------------
	9	SENSOR_TYPE_GRAVITY						重力传感器简称GV-sensor，输出重力数据。在地球上，重力数
		重力传感器								值为9.8，单位是m/s^2。坐标系统与加速度传感器相同。
												当设备复位时，重力传感器的输出与加速度传感器相同
	-----------------------------------------------------------------------------------------------
	10	SENSOR_TYPE_LINEAR_ACCELERATION			线性加速度传感器简称LA-sensor。  线性加速度传感器是
		线性加速度传感器							加速度传感器减去重力影响获取的数据。  单位是m/s^2，
												坐标系统与加速度传感器相同。 加速度传感器、重力传感器和
												线性加速度传感器的计算公式如下：  
												加速度 = 重力 + 线性加速度
	-----------------------------------------------------------------------------------------------
	11	SENSOR_TYPE_ROTATION_VECTOR				旋转矢量传感器简称RV-sensor。旋转矢量代表设备的方向，
		旋转矢量传感器								是一个将坐标轴和角度混合计算得到的数据
	-----------------------------------------------------------------------------------------------
	12	SENSOR_TYPE_RELATIVE_HUMIDITY			湿度传感器测量周围空气湿度，传感器返回的只是一下湿度
		湿度传感器								的百分比，要确定具体的湿度还需要其他sensor的配合
	-----------------------------------------------------------------------------------------------
	13	SENSOR_TYPE_AMBIENT_TEMPERATURE			环境（室）在摄氏度的温度
		环境温度传感器
	-----------------------------------------------------------------------------------------------




## Android Sensor 层次结构
Sensor 框架分为三个层次，客户度、服务端、HAL层，服务端负责从HAL读取数据，并将数据写到管道中，客户端通过管道读取服务端数据。

![Alt text](http://zhongguomin.github.io/blog/media/images/2014/Anddroid-Sensor-01.jpg "Anddroid-Sensor-01.jpg")


1	客户端主要类

SensorManager.java		
从android4.1开始，把SensorManager定义为一个抽象类，定义了一些主要的方法，类主要是应用层直接使用的类，提供给应用层的接口

SystemSensorManager.java		
继承于SensorManager，客户端消息处理的实体，应用程序通过获取其实例，并注册监听接口，获取sensor数据

sensorEventListener接口		
用于注册监听的接口

sensorThread		
是SystemSensorManager的一个内部类，开启一个新线程负责读取读取sensor数据，当注册了sensorEventListener接口的时候才会启动线程

android_hardware_SensorManager.cpp		
负责与java层通信的JNI接口

SensorManager.cpp		
sensor在Native层的客户端，负责与服务端SensorService.cpp的通信

SenorEventQueue.cpp		
消息队列


2	服务端主要类

SensorService.cpp		
服务端数据处理中心

SensorEventConnection		
从BnSensorEventConnection继承来，实现接口ISensorEventConnection的一些方法，ISensorEventConnection在SensorEventQueue会保存
一个指针，指向调用服务接口创建的SensorEventConnection对象
	
Bittube.cpp		
在这个类中创建了管道，即共享内存，用于服务端与客户端读写数据

SensorDevice		
负责与HAL读取数据




## 具体流程分析

- 客户端获取数据（From Server）

apk注册监听器		
	Activity实现了SensorEventListener接口，在onCreate方法中，获取SystemSensorManager，并获取到加速传感器的Sensor，
	在onResume方法中调用SystemSensorManager. registerListenerImpl注册监听器，当Sensor数据有改变的时候将会回调onSensorChanged方法。

初始化SystemSensorManager		
	系统开机启动的时候，会创建SystemSensorManager的实例，在其构造函数中，主要做了四件事情

初始化JNI		
	调用JNI函数nativeClassInit()进行初始化

初始化Sensor列表			
	调用JNI函数sensors_module_init，对Sensor模块进行初始化。创建了native层SensorManager的实例。

获取Sensor列表		
	调用JNI函数sensors_module_get_next_sensor()获取Sensor，并存在sHandleToSensor列表中

构造SensorThread类		
	构造线程的类函数，并没有启动线程，当有应用注册的时候才会启动线程

启动SensorThread线程读取消息队列中数据		
	当有应用程序调用registerListenerImpl()方法注册监听的时候，会调用SensorThread.startLoacked()启动线程,线程只会启动一次，
	并调用enableSensorLocked()接口对指定的sensor使能，并设置采样时间

在open函数中调用JNI函数sensors_create_queue()来创建消息队列,然后调用SensorManager. createEventQueue()创建。
	
在startLocked函数中启动新的线程后，做了一个while的等待while (mSensorsReady == false)，只有当mSensorsReady等于true的时候，
才会执行enableSensorLocked()函数对sensor使能。而mSensorsReady变量，是在open()调用创建消息队列成功之后才会true



- 服务端获取数据（From HAL）

启动SensorService服务		
	在SystemServer进程中的main函数中，通过JNI调用，调用到
	com_android_server_SystemServer.cpp的android_server_SystemServer_init1()方法，该
	方法又调用system_init.cpp中的system_init():

SensorService初始化		
	SensorService创建完之后，将会调用SensorService::onFirstRef()方法，在该方法中完成初始化工作。
	首先获取SensorDevice实例，在其构造函数中，完成了对Sensor模块HAL的初始化：
	
这里主要做了三个工作：		
	调用HAL层的hw_get_modele()方法，加载Sensor模块so文件
	调用sensor.h的sensors_open方法打开设备
	调用sensors_poll_device_t->activate()对Sensor模块使能
	
再来看看SensorService::onFirstRef()方法：在这个方法中，主要做了4件事情：			
	创建SensorDevice实例
	获取Sensor列表
	调用SensorDevice.getSensorList(),获取Sensor模块所有传感器列表
	为每个传感器注册监听器
	启动线程读取数据
	调用run方法启动新线程，将调用SensorService::threadLoop()方法。
	
在新的线程中读取HAL层数据			
	SensorService实现了Thread类，当在onFirstRef中调用run方法的后，将在新的线程中调用SensorService::threadLoop()方法。




- 服务端与客户端通信

客户端服务端线程			
	在图中我们可以看到有两个线程，一个是服务端的一个线程，这个线程负责源源不断的从HAL读取数据。另一个是客户端的一个线程，
客户端线程负责从消息队列中读数据。

创建消息队列			
客户端可以创建多个消息队列，一个消息队列对应有一个与服务器通信的连接接口

创建连接接口			
服务端与客户端沟通的桥梁，服务端读取到HAL层数据后，会扫面有多少个与客户端连接的接口，然后往每个接口的管道中写数据

创建管道			
每一个连接接口都有对应的一个管道。

上面是设计者设计数据传送的原理，但是目前Android4.1上面的数据传送不能完全按照上面的理解。因为在实际使用中，消息队列只会创建一个，
也就是说客户端与服务端之间的通信只有一个连接接口，只有一个管道传数据。那么数据的形式是怎么从HAL层传到JAVA层的呢？其实数据是以
一个结构体sensors_event_t的形式从HAL层传到JNI层。看看HAL的sensors_event_t结构体：

在JNI层有一个ASensorEvent结构体与sensors_event_t向对应，

在JNI层，只会将结构体数据中一部分的信息传到JAVA层：

经过前面的介绍，我们知道了客户端实现的方式及服务端的实现，但是没有具体讲到它两是如何进行通信的，这节我们专门介绍客户端与服务端
之间的通信。

这里主要涉及的是进程间通信，有IBind和管道通信。客户端通过IBind通信获取到服务端的远程调用，然后通过管道进行sensor数据的传输。



native层实现了sensor服务的核心实现，Sensor服务的主要流程的实现在sensorservice类中，

继承BinderService<SensorService>这个模板类添加到系统服务,用于Ibinder进程间通信。

在前面的介绍中，SensorService服务的实例是在System_init.cpp中调用SensorService::instantiate()创建的，即调用了上面的instantiate()
方法，接着调用了publish(),在该方法中，我们看到了new SensorService的实例，并且调用了defaultServiceManager::addService()将Sensor
服务添加到了系统服务管理中，客户端可以通过defaultServiceManager:getService()获取到Sensor服务的实例。

createSensorEventConnection()方法，该在服务端被实现，在客户端被调用，并返回一个SensorEventConnection的实例，创建连接，客户端拿到SensorEventConnection实例之后，可以对sensor进行通信操作，仅仅作为通信的接口而已，它并没有用来传送Sensor数据，因为Sensor数据量比较打，IBind实现比较困难。真正实现Sensor数据传送的是管道，在创建SensorEventConnection实例中，创建了BitTube对象，里面创建了管道，用于客户端与服务端的通信




## 附一 参考资料			
1	Sensor Framework原理			
	http://blog.csdn.net/lizzywu/article/details/10226099		
	注意，可以详细了解相关链接文章		
	深入浅出 - Android系统移植与平台开发（十四） - Sensor HAL框架分析之四			
2	android sensor framework			
	http://blog.csdn.net/tommy_wxie/article/details/13773597			

3	Android如何实现振动器的移植与开发		
	http://www.androidcn.com/news/20110424/00001586.html		
4	梳理一下传感器的数据流和框架是怎么样让屏幕旋转的。		
	http://blog.csdn.net/a345017062/article/details/6592527		

