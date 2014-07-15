---
layout: post
title: Android Sensor
categories:
- programmer
tags:
- android
---


## 问题


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


## Android sensor构建
Android实现传感器系统包括以下几个部分：		
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



## 附一 参考资料			
2	Android Sensor框架HAL层解读		
	http://www.cnblogs.com/lcw/p/3402816.html		
4	Android感应检测Sensor(简单介绍)		
	http://blog.csdn.net/huangbiao86/article/details/6745933	


3	A​n​d​r​o​i​d​ ​S​e​n​s​o​r​s​分​析		
	http://wenku.baidu.com/view/c54ed6e09b89680203d825db.html		
7	android sensor framework			
	http://blog.csdn.net/tommy_wxie/article/details/13773597		
8	Android Sensor传感器系统架构初探			
	http://blog.csdn.net/qianjin0703/article/details/5942579		
\
		

