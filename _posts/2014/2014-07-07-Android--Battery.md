---
layout: post
title: Android Battery
categories:
- programmer
tags:
- android
---


基于 android 4.0


## 问题
android进入和退出电池充电的下层接口和流程			
USB OTG Host时，DC IN 无法充电


## 锂电池基本介绍

1	简述工作原理			
目前锂电池公认的基本原理是所谓的“摇椅理论”。锂电池的冲放电不是通过传统的方式实现电子的转移，
而是通过锂离子在层壮物质的晶体中的出入，发生能量变化。在正常冲放电情况下，锂离子的出入一般
只引起层间距的变化，而不会引起晶体结构的破坏，因此从冲放电反映来讲，锂离子电池是一种理想的
可逆电池。在冲放电时锂离子在电池正负极往返出入，正像摇椅一样在正负极间摇来摇去，故有人将锂
离子电池形象称为摇椅电池


2	锂电池的充电方式是限压横流方式						
第一步：判断电压<3V，要先进行预充电，0.05C电流				
第二步：判断 3V<电压<4.2V，恒流充电0.2C~1C电流			
第三步：判断电压>4.2V,恒压充电,电压为4.20V,电流随电压的增加而减少，直到充满

术语解释：			
充放电电流一般用C作参照，C是对应电池容量的数值。电池容量一般用Ah、mAh表示，如M8的电池容量1200mAh，
对应的C就是1200mA。0.2C就等于240mA




## 关机充电（AC）

1	正常开机流程			
可大致分成三部分			
1)	OS_level: uboot、kernel、init这三步完成系统启动			
	bootloader -> linux kernel -> init
2)	Android_level: 这部分完成android部的初始化					
	init.rc -> servicemanager -> zygote -> systemserver
3)	Home Screen: 这部分就是我们看到的launcher部分				
	launcher


2	关机充电启动流程			
1)	插入 DC					
2)	启动系统，进入uboot，设置 androidboot.mode=charger						
3)	启动 kermel				
4)	启动 androdi，进入 init.c				
5)	判断 androidboot.mode，如果是 charger，做相应处理，显示充电图标			
6)	如果长按开机键，正常开机					
7)	如果屏幕超时，关闭充电图标显示			


3	关机充电软件逻辑
DC插入，其实相当于关机状态下“按开机键”开机。第一步要走UBOOT、kernel 、android init这一流程

1)	如何判断 DC 插入？
2)	设定 setenv("bootargs", "androidboot.mode=charger")
	androidboot.mode这个参数相当重要，这个参数决定系统是正常启动、还是关机充电状态

----------------------------------------------
	检查 DC 插入代码


boot-v1.0/u-boot/drivers/power/twl6030.c
	twl6030_stop_usb_charging
	twl6030_start_usb_charging
	twl6030_init_battery_charging


linux-3.3/drivers/power/charger-manager.c
linux-3.3/drivers/power/axp_power/axp22-sply.c


初步的理解
	关机状态下，插入DC，相当于按开机键，系统会走 uboot，kernel，android init 流程
	只是在 uboot 中，会判断这是DC插入，因而设置 androidboot.mode=charger，接着 kernel
	正常启动，然后是 android init，在 init 启动中，会读取 androidboot.mode，如果是 charger 
	状态，做相应的处理，比如跳过文件系统挂载，跳转至充电状态，显示充电图标。
	这里，会获取当前电池状态，显示对应的充电图标状态，并处理按键事件，屏幕超时事件。
	这是应用层的处理

	uboot中判断是 DC 插入后，应该不只是设置 androidboot.mode，应该还会触发充电动作。

	现在的关键是：没有找到判断 DC 插入的地方。
	以及如何判断是AC，还是 USB ？

----------------------------------------------




	android init 启动
	在UBOOT中把androidboot.mode设定为charger状态，内核正常流程启动，然后到init时要对charger这种状态处理
	
	system/core/init/init.c
		is_charger = !strcmp(bootmode, "charger");

		if (!is_charger)
			property_load_boot_defaults();

		这里就是UBOOT中设定的bootmode，如果是charger模式，跳过下面初始化
		/* skip mounting filesystems in charger mode */
		if (!is_charger) {
			action_for_each_trigger("early-fs", action_add_queue_tail);
			queue_builtin_action(console_init_action, "console_init");
			action_for_each_trigger("fs", action_add_queue_tail);
			action_for_each_trigger("post-fs", action_add_queue_tail);
			action_for_each_trigger("post-fs-data", action_add_queue_tail);
		}

		如果为charger，则调用charger.c
		if (is_charger) {
			action_for_each_trigger("charger", action_add_queue_tail); 
		} else {
			action_for_each_trigger("early-boot", action_add_queue_tail);
			action_for_each_trigger("boot", action_add_queue_tail); 
		}



	这部分就是我们充电部分，充电画面显示的实现

	system/core/charger/charger.c
		创建/sys/class/power_supply结点，把socket信息通知应用层
		uevent_open_socket这个函数是通过kobject_uevent的方式通知的应用层，就是往一个socket广播一个消息，
		只需要在应用层打开socket监听NETLINK_KOBJECT_UEVENT组的消息，就可以收到了,主要是创建了socket接口
		获得uevent的文件描述符，然后触发/sys/class/power_supply目录及其子目录下的uevent，然后接受并创建
		设备节点，至此设备节点才算创建
		coldboot(charger, "/sys/class/power_supply", "add"); 
		
		event_loop循环，电池状态，检测按键是否按下
		这个函数判断按键状态，DC是否插拔。如果长按开机：执行android_reboot(ANDROID_RB_RESTART,0, 0);
		如果拔出DC:执行android_reboot(ANDROID_RB_POWEROFF,0, 0);
		event_loop(charger);

			检查按键状态
			handle_input_state(charger, now);

			检查DC是否拔出
			handle_power_supply_state(charger, now); 

			对按键时间状态标志位的判断，显示不同电量的充电logo;
			update_screen_state(charger, now);
		





应用层 - 电池系统
1	电池系统结构
	Android中的电池使用方式主要有三种：AC、USB、Battery 等不同的模式

	在应用程序层次,通常包括了电池状态显示的功能。因此从 Android 系统的软件方面
	(包括驱动程序和用户空间内容)需要在一定程度上获得电池的状态，电池系统主要负责电池信息统计、显示


电池系统的架构如下所示
	framework
		BatteryStatus
		BatteryManager
		BatteryService
	JNI
		com_android_server_BatteryService.cpp
	Driver
		linux3.4/drivers/power/power_supply


Driver
	特定硬件平台电池的驱动程序,用 Linux的Power Supply 驱动程序,实现向用户空间提供信息。Battery 驱动程序需要
	通过sys文件系 统向用户空间提供接口, sys文件系统的路径是由上层的程序指定的。Linux标准的 Power Supply驱动程序 
	所使用的文件系统路径为:/sys/class/power_supply ,其中的每个子目录表示一种能源供应设备的名称。

	Power Supply 驱动程序的头文件在
		include/linux/power_supply.h
	中定义

	Linux中驱动程序：power_supply


JNI
	代码路径: 
		frameworks/base/services/jni/com_android_server_BatteryService.cpp 
	这个类调用sys文件系统访问驱动程序,也同时提供了JNI的接口

	处理的流程为根据设备类型判定设备后, 得到各个设备的相关属性,则需要得到更多得 信息。
	例如:果是交流或者 USB 设备,只需 要得到它们是否在线( onLine );如果是电 池设备,则需要得到更多的信息,
	例如状态 ( status ),健康程度( health ),容 量( capacity ),电压 ( voltage_now )等

	Linux 驱动 driver 维护着保存电池信息的一组文件 sysfs,供应用程序获取电源相关状态


Framework
		
		      BatteryService (java) ------>  ACTION_BATTERY_CHANGED
		        /           |                        |
		  read / battery    |                        |
		      / info        | uevent                 V
		     /              |               application hander boradcast msg
		Battery JNI         | power_supply
		      \             |
		 write \ battery    |
		        \ info      |
		         \          |
		         Battery Driver


	BatteryService 
	作为电池及充电相关的服务: 监听 Uevent、读取sysfs 里中的状态 、广播Intent.ACTION_BATTERY_CHANGED。

1)	mUEventObserver
	BatteryService实现了一个UevenObserver mUEventObserver。
	uevent是Linux 内核用来向用户空间主动上报事件的机制,对于JAVA程序来说,只实现 UEventObserver的虚函数 onUEvent,然后注册即可。
	BatteryService只关注 power_supply 的事件,所以在构造函数注册

2)	update()
	update读取sysfs文件做到同步取得电池信息, 然后根据读到的状态更新 BatteryService 的成员变量,
	并广播一个Intent来通知其它关注电源状态的 组件

	当kernel有power_supply事件上报时, mUEventObserver调用update()函数,然后update 调用native_update从
	sysfs中读取相关状态(com_android_server_BatteryService.cpp):

3)	sysfs
	Linux 驱动 driver 维护着保存电池信息的一组文件 sysfs,供应用程序获
	取电源相关状态:


数据来源
	BatteryService通过JNI（com_android_server_BatteryService.cpp）读取数据
数据传送
	 BatteryService主动把数据传送给所关心的应用程序，所有的电池的信息数据是通过Intent传送出去的
数据接收
	应用如果想要接收到BatteryService发送出来的电池信息，则需要注册一个Intent为Intent.ACTION_BATTERY_CHANGED
	的BroadcastReceiver
数据更新
	电池的信息会随着时间不停变化，自然地，就需要考虑如何实时的更新电池的数据信息。在BatteryService启动的时候，
	会同时通过UEventObserver启动一个onUEvent Thread
	在UEvent thread中会不停调用 update()方法，来更新电池的信息数据





















## 附一 参考资料
1	android 电池（一）：锂电池基本原理篇								
	http://blog.csdn.net/xubin341719/article/details/8497830		
2	android 电池（二）：android关机充电流程、充电画面显示				
	http://blog.csdn.net/xubin341719/article/details/8498580		
3	android 电池（三）：android电池系统								
	http://blog.csdn.net/xubin341719/article/details/8709838		
4	android电池（四）：电池 电量计(MAX17040)驱动分析篇				
	http://blog.csdn.net/xubin341719/article/details/8969369		
5	android电池（五）：电池 充电IC（PM2301）驱动分析篇				
	http://blog.csdn.net/xubin341719/article/details/8970363		

