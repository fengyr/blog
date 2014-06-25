---
layout: post
title: Android--GPS
categories:
- programmer
tags:
- android
---


## 一 Android GPS 基本介绍


## 二 Android GPS 框架
> 1	基本框架

		Application
			LOcation Application
		-------------------------------------
		Framework
			LocationManagerService
			LocationManager
			LocationProvider
			LocationListener
		-------------------------------------
		JNI
			com_android_server_location
				_GpsLocationProvider
		-------------------------------------
		HAL
			gps_device_t
			GpsInterface
		-------------------------------------
		Android Kernel
			GPS Driver
		-------------------------------------
				GPS Device



> 2	涉及文件
		frameworks/base/location/
		frameworks/base/services/java/com/android/server/location/
		frameworks/base/services/java/com/android/server/LocationManagerService.java
		com_android_server_location_GpsLOcationProvider.cpp
		hardware/qcom/gps/
		gps.h



## 三 Android GPS 具体流程
> 1	初始化


> 2	获取位置信息


## 四 参考资料
1	Android系统Gps分析（一）										
	http://blog.csdn.net/xnwyd/article/details/7198728			
2	基于android 的GPS 移植——主要结构体及接口介绍					
	http://blog.csdn.net/jshazk1989/article/details/6876410		
3	基于android 的GPS 移植——调用关系								
	http://blog.csdn.net/jshazk1989/article/details/6877823		



