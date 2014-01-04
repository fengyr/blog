---
layout: post
title: ubuntu下搭建Android开发环境
categories:
- programmer
tags:
- android_basis
---


###前言
一直以来，没有认真整理过android开发环境的搭建过程，虽然已经搭建过几次。
每次搭建时，都是上网查下资料，东看点，西看点，很是浪费时间。

在这里，整理下主要的步骤

###1	安装JDK
下载JDK安装包，解压	
下载地址

	http://www.oracle.com/technetwork/cn/java/javase/downloads/jdk7-downloads-1880260-zhs.html
配置环境变量
验证安装

	java -version


###2	安装 Eclipse
下载Eclipse压缩包，解压即可

	http://www.eclipse.org/downloads/


###3	下载Android SDK
下载后解压
eclispe中设置android sdk 路径
设置好后，打开Android SDK Manager，选择下载tools和需要的android版本

这样下载 android sdk那是相当的慢啊！

两种解决办法

- 1 下载多合一安装包
下载文件与地址

	adt-bundle-linux-x86_64-20131030.zip
	http://developer.android.com/sdk/index.html

看下关于adt-bundle说明

	With a single download, the ADT Bundle includes everything you need to begin developing apps:
		Eclipse + ADT plugin
		Android SDK Tools
		Android Platform-tools
		The latest Android platform
		The latest Android system image for the emulator

解压后，得到这样的目录结构

	adt-bundle-linux-x86_64-20131030
	├── eclipse
	└── sdk

看到目录，应该知道怎么回事了。

- 2 分步下载开发包
参考文章 

	Android SDK开发包国内下载地址
	http://www.cnblogs.com/bjzhanghao/archive/2012/11/14/android-platform-sdk-download-mirror.html
	
上面有android开发需要的各种资源包，包括ADT Bundle, Android SDK, ADT, Platforms, Documents, Sources, Samples
必须下载的资源包有：

	Android SDK(android-sdk_r22.3-linux.tgz platform-tools_r14-linux.zip)
	Platforms(android-17_r01.zip sysimg_armv7a-17_r01.zip)


###4	安装ADT
	eclipse中安装ADT插件


###5	完成，可以新建AVD，android工程了




	
