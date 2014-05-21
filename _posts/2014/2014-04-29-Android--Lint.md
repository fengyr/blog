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



##问题描述
Android-Lint所要检查的问题以Issue来描述。


##参考资料
lint	
http://developer.android.com/tools/help/lint.html

Improving Your Code with lint		
http://developer.android.com/tools/debugging/improving-w-lint.html

Android-Lint：查错与代码优化利器		
http://blog.csdn.net/thl789/article/details/8037473

定制Android-Lint检查问题的现有规则		
http://blog.csdn.net/thl789/article/details/8036066

提高你的代码稳定性与可读性-lint工具		
http://www.cnblogs.com/wanqieddy/p/3549739.html

