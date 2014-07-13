---
layout: post
title: Android-GPS
categories:
- programmer
tags:
- android
---


## 一 Android GPS 基本介绍
1.	GPS 介绍

什么是 GPS		
GPS是英文 Global Positioning System（全球定位系统）的简称		
利用GPS定位卫星，在全球范围内实时进行定位、导航的系统，称为全球卫星定位系统，简称GPS

有什么功能		
可以在全球范围内实现全天候、连续、实时的三维导航定位和测速

原理是什么		
GPS定位的基本原理是根据高速运动的卫星瞬间位置作为已知的起算数据，采用空间距离后方交会的方法，确定待测点的位置


2.	Android GPS 介绍		




## 二 Android GPS 框架

基于 __android4.0__

1.	基本框架

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


2.	几个重要的地方		
1)	大概的流程，逻辑简述

2）	主要的几个线程

3）	几个重要的流程



3.	涉及文件

		frameworks/base/location/
		frameworks/base/services/java/com/android/server/location/
		frameworks/base/services/java/com/android/server/LocationManagerService.java
		com_android_server_location_GpsLOcationProvider.cpp
		hardware/qcom/gps/
		gps.h


4.	GPS 头文件定义		
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


5.	HAL		
GPS硬件适配层的源码位于：			
hardware/qcom/gps目录下			

重要的几个地方		
- xxxx
- xxxx



6.	JNI		
GPS JNI适配层的源码位于：		
frameworks/base/services/jni/com_android_server_location_GpsLocationProvider.cpp			

重要的几个地方		
- xxxx
- xxxx



7.	Framework		
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
	

8.	Application			




## 三 Android GPS 具体流程
1.	初始化		
从 SystemServer 启动 LocationManagerService, 到 Framework, JNI, HAL 的过程

	SystemServer.java
		location = new LocationManagerService(context);
		ServiceManager.addService(Context.LOCATION_SERVICED, location);
	LocationManagerService.java
		run
			mLocationHandler = new LocationWorkerHandler();
				handler msg: MESSAGE_LOCATION_CHANGED MESSAGE_PACKAGE_UPDATED

			initialize() -> loadProviders() -> ... -> _loadProvidersLocked()
				if GpsLocationProvider.isSupported
					class_init_native -> JNI
						android_location_GpsLocationProvider_class_init_native
						1)	get method at GpsLocationProvider
								env->GetMethodID(... method_name)
						2)	get GpsInterface sGpsInterface
								hw_get_module(.. module)
								module->methods->open(... device)
								sGpsInterface = gps_device->get_gps_interface(gps_device);
						3)	get GpsXXXInterface, AGpsXXXInterface
								sGpsInterface->get_extension

		detail for 2)
			see at
				hardware/qcom/gps/loc_api/libloc_api/gps.c
				hardware/qcom/gps/loc_api/libloc_api/loc_eng.cpp
				hardware/libhardware/include/hardware/gps.h
				hardware/libhardware/hardware.c

		detail for 3)
			see at 
				hardware/qcom/gps/loc_api/libloc_api/loc_eng.cpp
				hardware/libhardware/include/hardware/gps.h
			


2.	获取位置信息

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


从 Application 如上代码 ( __locationManager.getLastKnownLocation__ ) 到 Framework, JNI, HAL 的过程		

__locationManager.getBestProvider__
__locationManager.getLastKnownLocation__






3.	位置改变

	//匿名类，继承自LocationListener接口
	private final LocationListener locationListener = new LocationListener() {
		public void onLocationChanged(Location location) {
			updateWithNewLocation(location);		//更新位置信息
		}

		public void onProviderDisabled(String provider){
			updateWithNewLocation(null);			//更新位置信息
		}

		public void onProviderEnabled(String provider){ }
		public void onStatusChanged(String provider, int status,Bundle extras){ }
    };

	//更新位置信息
	private void updateWithNewLocation(Location location) {
		if (location != null) {
			//获取经纬度
			double lat = location.getLatitude();
			double lng = location.getLongitude();
		}
	}
	
	//添加侦听
	locationManager.requestLocationUpdates(provider, 2000, 10,locationListener);


__locationManager.requestLocationUpdates__







4.	xxxxx





## 四 Android GPS DEMO APK





## 附一 参考资料
1.	Android系统Gps分析（一）										
	http://blog.csdn.net/xnwyd/article/details/7198728			
2.	基于android 的GPS 移植——主要结构体及接口介绍					
	http://blog.csdn.net/jshazk1989/article/details/6876410		
3.	基于android 的GPS 移植——调用关系								
	http://blog.csdn.net/jshazk1989/article/details/6877823		
4.	android developer		
	http://developer.android.com/reference/android/location/LocationManager.html			
5.	android GPS定位系统		
	http://www.2cto.com/kf/201205/129836.html				
6.	Android Gps  			
	http://xxw8393.blog.163.com/blog/static/3725683420107424543609/				
7	二十七、Android之GPS定位详解		
	http://www.cnblogs.com/linjiqin/archive/2011/11/01/2231598.html			
8	Android GPS应用：临近警告		
	http://blog.csdn.net/u010142437/article/details/9391189			
9	Android GPS应用：动态获取位置信息		
	http://blog.csdn.net/u010142437/article/details/9390663			
10	Android中GPS简介及其应用		
	http://blog.csdn.net/u010142437/article/details/9390065		



