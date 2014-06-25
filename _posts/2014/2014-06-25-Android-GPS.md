---
layout: post
title: Android--GPS
categories:
- programmer
tags:
- android
---


## 一 Android GPS 基本介绍
1	GPS 介绍		
GPS是英文Global Positioning System（全球定位系统）的简称		





## 二 Android GPS 框架

基于 android4.0

> 1	基本框架

		Application
			Location Application
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


> 3	GPS 头文件定义		
gps.h 定义了GPS底层相关的结构体和接口，其中几个重要的结构体		

	GpsLocation			
	GPS位置信息结构体，包含经纬度，高度，速度，方位角等		

	GpsStatus		
	GPS状态包括5种状态，分别为未知，正在定位，停止定位，启动未定义，未启动		

	GpsSvInfo		
	GPS卫星信息，包含卫星编号，信号强度，卫星仰望角，方位角等			

	GpsSvStatus		
	GPS卫星状态，包含可见卫星数和信息，星历时间，年历时间等		

	GpsCallbacks		
	回调函数定义		

	GpsInterface		
	GPS接口是最重要的结构体，上层是通过此接口与硬件适配层交互的		

	gps_device_t		
	GPS设备结构体，继承自hw_device_tcommon，硬件适配接口，向上层提供了重要的get_gps_interface接口		


> 4	HAL		
GPS硬件适配层的源码位于：hardware/qcom/gps目录下			


> 5	JNI		
GPSJNI适配层的源码位于：frameworks/base/services/jni/com_android_server_location_GpsLocationProvider.cpp			



> 6	Framework		
GPSFramework源码位于：frameworks/base/location		
GPSFramework重要的接口和类		

	GpsStatus.Listener
	用于当Gps状态发生变化时接收通知

	GpsStatus.NmeaListener
	用于接收Gps的NMEA数据

	LocationListener
	用于接收当位置信息发生变化时，LocationManager发出的通知


	Address				地址信息类
	Criteria			用于根据设备情况动态选择provider
	Geocoder			用于处理地理编码信息
	GpsSatellite		用于获取当前卫星状态
	GpsStatus			用于获取当前Gps状态
	Location			地理位置信息类
	LocationManager		用于获取和操作gps系统服务
	LocationProvider	抽象类，用于提供位置提供者
	

> 7	Application			




## 三 Android GPS 具体流程
> 1	初始化		

	SystemServer.java -> LocationManagerService.java -> ...

从 SystemServer 启动 LocationManagerService, 到 Framework, JNI, HAL 的过程



> 2	获取位置信息		

	//获取位置服务
	LocationManager locationManager = (LocationManager) getSystemService(Context.LOCATION_SERVICE);  

	Criteria criteria = new Criteria();  
	// 获得最好的定位效果  
	criteria.setAccuracy(Criteria.ACCURACY_FINE);			//设置为最大精度
	criteria.setAltitudeRequired(false);					//不获取海拔信息
	criteria.setBearingRequired(false);						//不获取方位信息
	criteria.setCostAllowed(false);							//是否允许付费
	criteria.setPowerRequirement(Criteria.POWER_LOW);		// 使用省电模式

	// 获得当前的位置提供者  
	String provider = locationManager.getBestProvider(criteria, true);
  
	// 获得当前的位置  
	Location location = locationManager.getLastKnownLocation(provider);  

	Geocoder gc = new Geocoder(this);   
	List<Address> addresses = null;  
	try {
		//根据经纬度获得地址信息  
		addresses = gc.getFromLocation(location.getLatitude(), location.getLongitude(), 1);  
	} catch (IOException e) {  
		e.printStackTrace();  
	}

	if (addresses.size() > 0) {
		//获取address类的成员信息
		Sring msg = “”;   
		msg += "AddressLine：" + addresses.get(0).getAddressLine(0)+ "\n";   
		msg += "CountryName：" + addresses.get(0).getCountryName()+ "\n";   
		msg += "Locality：" + addresses.get(0).getLocality() + "\n";   
		msg += "FeatureName：" + addresses.get(0).getFeatureName();   
	}


从 Application ( locationManager.getLastKnownLocation ) 到 Framework, JNI, HAL 的过程





## 四 参考资料
1	Android系统Gps分析（一）										
	http://blog.csdn.net/xnwyd/article/details/7198728			
2	基于android 的GPS 移植——主要结构体及接口介绍					
	http://blog.csdn.net/jshazk1989/article/details/6876410		
3	基于android 的GPS 移植——调用关系								
	http://blog.csdn.net/jshazk1989/article/details/6877823		
4	android developer		
	http://developer.android.com/reference/android/location/LocationManager.html			
5	android GPS定位系统		
	http://www.2cto.com/kf/201205/129836.html				
6	Android Gps  			
	http://xxw8393.blog.163.com/blog/static/3725683420107424543609/				



