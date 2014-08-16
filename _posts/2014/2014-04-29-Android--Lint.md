---
layout: post
title: Android--Lint
categories:
- programmer
tags:
- android
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

- Application source files
    The source files consist of files that make up your Android project, including Java and XML files, icons,
    and ProGuard configuration files.
- The lint.xml file
    A configuration file that you can use to specify any lint checks that you want to exclude and to customize
    problem severity levels.
- The lint tool
    A static code scanning tool that you can run on your Android project from the command-line or from Eclipse.
    The lint tool checks for structural code problems that could affect the quality and performance of your
    Android application.
    It is strongly recommended that you correct any errors that lint detects before publishing your application.
- Results of lint checking
    You can view the results from lint in the console or in the Lint Warnings view in Eclipse. Each issue is
    identified by the location
    in the source files where it occurred and a description of the issue.

The lint tool is automatically installed as part of the Android SDK Tools revision 16 or higher. If you want to use
lint in the Eclipse environment, you must also install the Android Development Tools (ADT) Plugin for Eclipse revision
16 or higher.
For more information about installing the SDK or the ADT Plugin for Eclipse, see Installing the SDK.

more about lint see at

    http://developer.android.com/tools/debugging/improving-w-lint.html#config


##问题描述
Android-Lint所要检查的问题以Issue来描述。		
Issue分9类（Category）：		
    Correctness/ Correctness: Messages / Security / Performance / Usability: Typography /		
	Usability: Icons / Usability / Accessibility / Internationalization。		
Issue以一个文本短语来作为id，对Issue的定制等操作都是基于id的。		
Issue以Severity来标识该Issue的危害程度：		
    Fatal / Error / Warning/ Information / Ignore。		
对Issue的忽略操作其实也就是降低它的Severity为Ignore。		


##参考资料
lint		
http://developer.android.com/tools/help/lint.html

Improving Your Code with lint			
http://developer.android.com/tools/debugging/improving-w-lint.html#config

Android-Lint：查错与代码优化利器		
http://blog.csdn.net/thl789/article/details/8037473

定制Android-Lint检查问题的现有规则		
http://blog.csdn.net/thl789/article/details/8036066

提高你的代码稳定性与可读性-lint工具		
http://www.cnblogs.com/wanqieddy/p/3549739.html

