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

	ID	Sensor Tyoe									Desc
	----------------------------------------------------------------------------------------------------
	1	SENSOR_TYPE_ACCELEROMETER					加速度传感器又叫G-sensor，返回x、y、z三轴的加速度数值。
		加速度传感器									该数值包含地心引力的影响，单位是m/s^2
	----------------------------------------------------------------------------------------------------
	2	SENSOR_TYPE_MAGNETIC_FIELD					磁力传感器简称为M-sensor，返回x、y、z三轴的环境磁场数据
		磁力传感器									该数值的单位是微特斯拉（micro-Tesla），用uT表示。  
													硬件上一般没有独立的磁力传感器，磁力数据由电子罗盘传感器
													提供（E-compass）。
	----------------------------------------------------------------------------------------------------
	3	SENSOR_TYPE_ORIENTATION						方向传感器简称为O-sensor，返回三轴的角度数据，方向数据
		方向传感器									的单位是角度。 为了得到精确的角度数据，E-compass需要获
													取G-sensor的数据，  经过计算生产O-sensor数据，否则只能
													获取水平方向的角度。  方向传感器提供三个数据，分别为
													azimuth、pitch和roll。
	----------------------------------------------------------------------------------------------------
	4	SENSOR_TYPE_GYROSCOPE						陀螺仪传感器叫做Gyro-sensor，返回x、y、z三轴的角速度
		陀螺仪										数据。  角速度的单位是radians/second
	----------------------------------------------------------------------------------------------------
	5	SENSOR_TYPE_LIGHT							光线感应传感器检测实时的光线强度，光强单位是lux，其物理
		光线传感器									意义是照射到单位面积上的光通量。  光线感应传感器主要用于
													Android系统的LCD自动亮度功能。  可以根据采样到的光强数
													值实时调整LCD的亮度
	----------------------------------------------------------------------------------------------------
	6	SENSOR_TYPE_PRESSURE						压力传感器返回当前的压强，单位是百帕斯卡hectopascal（hPa）
		压力传感器
	----------------------------------------------------------------------------------------------------
	7	SENSOR_TYPE_TEMPERATURE						温度传感器返回当前的温度
		温度传感器
	----------------------------------------------------------------------------------------------------
	8	SENSOR_TYPE_PROXIMITY						接近传感器检测物体与手机的距离，单位是厘米。
		近场传感器									接近传感器可用于接听电话时自动关闭LCD屏幕以节省电量
	----------------------------------------------------------------------------------------------------
	9	SENSOR_TYPE_GRAVITY							重力传感器简称GV-sensor，输出重力数据。在地球上，重力数
		重力传感器									值为9.8，单位是m/s^2。坐标系统与加速度传感器相同。
													当设备复位时，重力传感器的输出与加速度传感器相同
	----------------------------------------------------------------------------------------------------
	10	SENSOR_TYPE_LINEAR_ACCELERATION				线性加速度传感器简称LA-sensor。  线性加速度传感器是
		线性加速度传感器								加速度传感器减去重力影响获取的数据。  单位是m/s^2，
													坐标系统与加速度传感器相同。 加速度传感器、重力传感器和
													线性加速度传感器的计算公式如下：  
													加速度 = 重力 + 线性加速度
	----------------------------------------------------------------------------------------------------
	11	SENSOR_TYPE_ROTATION_VECTOR					旋转矢量传感器简称RV-sensor。旋转矢量代表设备的方向，
		旋转矢量传感器								是一个将坐标轴和角度混合计算得到的数据
	----------------------------------------------------------------------------------------------------
	12	SENSOR_TYPE_RELATIVE_HUMIDITY				湿度传感器测量周围空气湿度，传感器返回的只是一下湿度
		湿度传感器									的百分比，要确定具体的湿度还需要其他sensor的配合
	----------------------------------------------------------------------------------------------------
	13	SENSOR_TYPE_AMBIENT_TEMPERATURE				环境（室）在摄氏度的温度
		环境温度传感器
	----------------------------------------------------------------------------------------------------




## Android Sensor 层次结构

Java -> Framework -> JNI -> HAL -> Driver -> Device


- 重要的数据结构，函数


- Java


- Framework
SensorManager.java
		与下层接口功能：
		1) 在SensorManager函数中
		   (1) 调用native sensors_module_init初始化sensor list，即实例化native中的SensorManager
		   (2) 创建SensorThread线程
		2) 在类SensorThread中
		   (1) 调用native sensors_create_queue创建队列
		   (2) 在线程中dead loop地调用native sensors_data_poll以从队列sQueue中获取事件(float[] values = new float[3];)
		   (3) 收到事件之后，报告sensor event给所有注册且关心此事件的listener
		 
		与上层的接口功能：
		1) 在onPause时取消listener注册
		2) 在onResume时注册listener
		3) 把收到的事件报告给注册的listener

android_hardware_SensorManager.cpp
		实现SensorManager.java中的native函数，它主要调用SenrsorManager.cpp和SensorEventQueue.cpp中的类来完成相关的工作

SensorManager.cpp


SensorService.cpp
		SensorService主要功能如下：
          1) SensorService::instantiate创建实例对象，并增加到ServiceManager中，且创建并启动线程，并执行threadLoop
          2) threadLoop从sensor驱动获取原始数据，然后通过SensorEventConnection把事件发送给客户端
          3) BnSensorServer的成员函数负责让客户端获取sensor列表和创建SensorEventConnection


SensorDevice.cpp
		SensorDevice封装了对SensorHAL层代码的调用，主要包含以下功能：
         1) 获取sensor列表(getSensorList)
         2) 获取sensor事件(poll)
         3) Enable或Disable sensor (activate)
         4) 设置delay时间



- JNI


- HAL
Sensor HAL
		定义：/hardware/libhardware/include/hardware/sensors.h
		实现：/hardware/mychip/sensor/st/sensors.c

		struct sensors_poll_device_t 定义 
		struct sensors_module_t  定义
		struct sensor_t 定义
		struct sensors_event_t 定义

		struct sensors_module_t 实现
		struct sensors_poll_device_t 实现

		struct sensors_poll_context_t 定义 
		struct sensors_poll_context_t 的实现


- Driver






Android实现传感器系统包括以下几个部分
1	java层			Sensor API
2	framework层		SensorManager SensorService
3	JNI层			Sensor JNI
4	HAL层			Sensor HAL
5	驱动层			Sensor Driver


3.1	Sensor HAL层接口
	Google为Sensor提供了统一的HAL接口，不同的硬件厂商需要根据该接口来实现并完成具体的硬件抽象层。
	Android中Sensor的HAL接口定义在：hardware/libhardware/include/hardware/sensors.h
	包括：
		对传感器类型的定义
		传感器模块的定义结构体 sensors_module_t
			该接口的定义实际上是对标准的硬件模块hw_module_t的一个扩展，增加了一个get_sensors_list函数，用于获取传感器的列表
		对任意一个sensor设备都会有一个sensor_t结构体
		每个传感器的数据由sensors_event_t结构体表示
		sensors_vec_t结构体用来表示不同传感器的数据
		Sensor设备结构体sensors_poll_device_t，对标准硬件设备hw_device_t结构体的扩展，主要完成读取底层数据，
			并将数据存储在struct sensors_poll_device_t结构体中
		控制设备打开/关闭结构体 sensors_open sensors_close


3.2	Sensor HAL实现
	============================================================================================
	SensorDevice.cpp			hardware.c / .h			sensor.h		sensor.cpp		设备节点
	============================================================================================
	1 SensorDevice()
	2 hw_get_module()
								3 hw_get_module_by_class
								4 load()
								5 sensors_open()
														6 hw_module_t->hw_module_methods_t->open()
																		7 open_sensors()
																		8 new sensor_poll_context_t()
																		9 open_input_device()
																		10 open
																		11 ioctl()
	12 sensors_module_t->get_sensors_list()
														13 sensors_get_sensors_list()
								14 poll_activate()
																		15 set_sysfs_input_attr()
																		16 open()
																		17 write()
								18 poll()
														19 poll_poll()
																		20 open()
																		21 read()
	============================================================================================

	SensorDevice属于JNI层，与HAL进行通信的接口
	在JNI层调用了HAL层的open_sensors()方法打开设备模块，再调用poll__activate()对设备使能，然后调用poll__poll读取数据。




## 具体流程分析

- 获取传感器


- 打开设备


- 获取数据




## 涉及文件

	./frameworks/base/core/java/android/hardware/Sensor.java
	./frameworks/base/core/java/android/hardware/SensorManager.java
	./frameworks/base/core/java/android/hardware/SensorListener.java
	./frameworks/base/core/java/android/hardware/SensorEvent.java
	./frameworks/base/core/java/android/hardware/SensorEventListener2.java
	./frameworks/base/core/java/android/hardware/SensorEventListener.java

	./frameworks/native/services/sensorservice/SensorDevice.h
	./frameworks/native/services/sensorservice/SensorFusion.h
	./frameworks/native/services/sensorservice/SensorService.cpp
	./frameworks/native/services/sensorservice/SensorDevice.cpp
	./frameworks/native/services/sensorservice/SensorService.h
	./frameworks/native/services/sensorservice/SensorFusion.cpp
	./frameworks/native/services/sensorservice/SensorInterface.cpp
	./frameworks/native/services/sensorservice/SensorInterface.h

	./frameworks/native/libs/gui/SensorManager.cpp
	./frameworks/native/libs/gui/SensorEventQueue.cpp
	./frameworks/native/libs/gui/Sensor.cpp

	./frameworks/native/include/gui/Sensor.h
	./frameworks/native/include/gui/SensorManager.h
	./frameworks/native/include/gui/SensorEventQueue.h

	./hardware/ti/omap4xxx/camera/inc/SensorListener.h
	./hardware/ti/omap4xxx/camera/SensorListener.cpp

	./hardware/akm/AK8975_FS/libsensors/SensorBase.cpp
	./hardware/akm/AK8975_FS/libsensors/SensorBase.h

	./hardware/invensense/65xx/libsensors_iio/SensorBase.cpp
	./hardware/invensense/65xx/libsensors_iio/SensorBase.h
	./hardware/invensense/60xx/libsensors_iio/SensorBase.cpp
	./hardware/invensense/60xx/libsensors_iio/SensorBase.h
	./hardware/invensense/60xx/libsensors/SensorBase.cpp
	./hardware/invensense/60xx/libsensors/SensorBase.h

	./device/samsung/manta/libsensors/SensorBase.cpp
	./device/samsung/manta/libsensors/SensorBase.h

	./device/generic/goldfish/camera/fake-pipeline2/Sensor.h
	./device/generic/goldfish/camera/fake-pipeline2/Sensor.cpp

	./device/softwinner/fiber-common/hardware/libhardware/libsensors/SensorBase.cpp
	./device/softwinner/fiber-common/hardware/libhardware/libsensors/SensorBase.h





## 附一 参考资料			
1	Android Sensor框架HAL层解读		
	http://www.cnblogs.com/lcw/p/3402816.html		
2	Android感应检测Sensor(简单介绍)		
	http://blog.csdn.net/huangbiao86/article/details/6745933	
3	Android Sensor传感器系统架构初探			
	http://blog.csdn.net/qianjin0703/article/details/5942579		
4	A​n​d​r​o​i​d​ ​S​e​n​s​o​r​s​分​析		
	http://wenku.baidu.com/view/c54ed6e09b89680203d825db.html		
5	android sensor framework			
	http://blog.csdn.net/tommy_wxie/article/details/13773597		

		

