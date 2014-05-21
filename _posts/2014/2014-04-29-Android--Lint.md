---
layout: post
title: Android--Lint
categories:
- programmer
tags:
- tools
---


##lint
The Android lint tool is a static code analysis tool that checks your Android project source files for potential bugs 
and optimization improvements for correctness, security, performance, usability, accessibility, and internationalization.


##Syntax

	lint [flags] <project directory>


##Options
see at

	http://developer.android.com/tools/help/lint.html

describes the command-line options for lint.


##Improving Your Code with lint
In addition to testing that your Android application meets its functional requirements, it is important to ensure 
that your code has no structural problems. Poorly structured code can impact the reliability and efficiency of your 
Android apps and make your code harder to maintain. For example, if your XML resource files contain unused namespaces, 
this takes up space and incurs unnecessary processing. Other structural issues, such as use of deprecated elements 
or API calls that are not supported by the target API versions, might lead to code failing to run correctly.


##Overview
The Android SDK provides a code scanning tool called lint that can help you to easily identify and correct problems 
with the structural quality of your code, without having to execute the app or write any test cases. Each problem 
detected by the tool is reported with a description message and a severity level, so that you can quickly prioritize 
the critical improvements that need to be made. You can also configure a problem is severity level to ignore issues 
that are not relevant for your project, or raise the severity level. The tool has a command-line interface, so you 
can easily integrate it into your automated testing process.

The lint tool checks your Android project source files for potential bugs and optimization improvements for correctness,
security, performance, usability, accessibility, and internationalization. You can run lint from the command-line or 
from the Eclipse environment.

Figure 1 shows how the lint tool processes the application source files.

![Alt text](http://zhongguomin.github.io/blog/media/images/2014/lint_01.png "lint_01.png")

Figure 1. Code scanning workflow with the lint tool



##��������
Android-Lint��Ҫ����������Issue��������


##�ο�����
lint	
http://developer.android.com/tools/help/lint.html

Improving Your Code with lint		
http://developer.android.com/tools/debugging/improving-w-lint.html

Android-Lint�����������Ż�����		
http://blog.csdn.net/thl789/article/details/8037473

����Android-Lint�����������й���		
http://blog.csdn.net/thl789/article/details/8036066

�����Ĵ����ȶ�����ɶ���-lint����		
http://www.cnblogs.com/wanqieddy/p/3549739.html

