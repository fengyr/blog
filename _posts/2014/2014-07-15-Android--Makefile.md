---
layout: post
title: Android Makefile
categories:
- programmer
tags:
- android
---


## Android Makefile 基本语法

## Android Makefile 目录结构


首先我们来看看android里makefile的写法

(1)Android.mk文件首先需要指定LOCAL_PATH变量，用于查找源文件
	LOCAL_PATH:=$(call my-dir)


(2)Android.mk中可以定义多个编译模块，每个编译模块都是
以include $(CLEAR_VARS)开始
以include $(BUILD_XXX)结束。
	CLEAR_VARS由编译系统提供，指定让GNU MAKEFILE为你清除除LOCAL_PATH以外的所有LOCAL_XXX变量

	include $(BUILD_STATIC_LIBRARY)表示编译成静态库
	include $(BUILD_SHARED_LIBRARY)表示编译成动态库。
	include $(BUILD_EXECUTABLE)表示编译成可执行程序
	include $(BUILD_PACKAGE)
	

