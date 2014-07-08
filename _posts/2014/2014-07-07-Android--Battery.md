---
layout: post
title: Android Battery
categories:
- programmer
tags:
- android
---


android 4.0


## 问题
android进入和退出电池充电的下层接口和流程		
USB OTG Host时，DC IN 无法充电


## 锂电池基本原理篇

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



## 
1	正常开机流程			
可大致分成三部分			
（1）、OS_level：UBOOT、kenrel、init这三步完成系统启动			
（2）、Android_level：这部分完成android部的初始化				
（3）、Home Screen：这部分就是我们看到的launcher部分			


2	关机充电启动流程			
1)	插入 DC				
2)	启动系统，进入uboot，设置 androidboot.mode=charger				
3)	启动 kermel			
4)	启动 androdi，进入 init.c			
5)	判断 androidboot.mode，如果是 charger，做相应处理，显示充电图标			
6)	如果长按开机键，正常开机			
7)	如果屏幕超时，关闭充电图标显示			



如何判断 DC 插入？

















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

