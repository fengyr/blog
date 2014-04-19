---
layout: post
title: 2014-04-19-Android下的VOIP实现
categories:
- programmer
tags:
- android_basis
---


##前言
最新要做一个移动端视频通话软件，大致看了下现有的开源软件

##一) sipdroid
- 1）架构
sip协议栈使用JAVA实现，音频Codec使用skype的silk（Silk编解码是Skype向第三方开发人员和硬件制造商提供免版税认证(RF)的Silk宽带音频编码器）实现。
NAT传输支持stun server.	

- 2）优缺点：
NAT方面只支持STUN，无ICE框架，如需要完全实现P2P视频通话需要实现符合ICE标准的客户端,音频方面没看到AEC等技术，视频方面还不是太完善，
目前只看到调用的是系统自带的MediaRecorder，并没有自己的第三方音视频编解码库。

- 3）实际测试：
基于sipdroid架构的话，我们要做的工作会比较多，（ICE支持，添加回音消除，NetEQ等gips音频技术，添加视频硬件编解码codec.）,所以就不做测试了。


##二) imsdroid
- 1)架构：
基于doubango(Doubango 是一个基于3GPP IMS/RCS 并能用于嵌入式和桌面系统的开源框架。该框架使用ANSCI-C编写，具有很好的可移植性。
并且已经被设计成非常轻便且能有效的工作在低内存和低处理能力的嵌入式系统上。苹果系统上的idoubs功能就是基于此框架编写) .
音视频编码格式大部分都支持（H264(video)，VP8(video)，iLBC(audio),PCMA,PCMU,G722,G729）。NAT支持ICE（stun+turn）

- 2）效果实测
测试环境：公司局域网内两台机器互通，服务器走外网sip2sip
音频质量可以，但是AEC打开了还是有点回音（应该可以修复）。视频马赛克比较严重，延迟1秒左右。

- 3）优缺点
imsdroid目前来说还是算比较全面的，包括音视频编解码，传输（RTSP，ICE），音频处理技术等都有涉猎。doubango使用了webrtc的AEC技术，
但是其调用webrtc部分没有开源，是用的编译出来的webrtc的库。如果要改善音频的话不太方便，Demo的音频效果可以，视频效果还是不太理想。


##三）csipsimple
- 1）sip协议栈用的是pjsip,音视频编解码用到的第三方库有ffmpeg（video）,silk(audio),webrtc.默认使用了webrtc的回声算法。支持ICE协议。

- 2）优缺点：
csipsimple架构比较清晰，sip协议由C实现，java通过JNI调用，SIP协议这一块会比较高效。其VOIP各个功能也都具备，包括NAT传输，音视频编解码。
并且该项目跟进新技术比较快，官方活跃程度也比较高。如果做二次开发可以推荐这个。

- 3）实测效果
测试环境：公司局域网内两台机器互通，服务器走外网sip2sip
音频质量可以，无明显回音，视频需要下插件，马赛克比imsdroid更严重。


##四）Linphone
这个是老牌的sip，支持平台广泛 windows, mac,ios,android,linux，技术会比较成熟。但是据玩过的同事说linphone在Android上的bug有点多，
由于其代码实在庞大，所以我暂时放弃考虑Linphone.不过如果谁有跨平台的需要，可以考虑Linphone或者imsdroid和下面的webrtc.。。。
好像现在开源软件都跨平台了。。。


##五) webrtc
imsdroid,csipsimple,linphone都想法设法调用webrtc的音频技术，本人也测试过Android端的webrtc内网视频通话，效果比较满意。
但是要把webrtc做成一个移动端的IM软件的话还有一些路要走，不过webrtc基本技术都已经有了，包括p2p传输，音视频codec,音频处理技术。
不过其因为目前仅支持VP8的视频编码格式（QQ也是）想做高清视频通话的要注意了。VP8在移动端的硬件编解码支持的平台没几个（RK可以支持VP8硬件编解码）。
不过webrtc代码里看到可以使用外部codec,这个还是有希望调到H264的。


##总结：
sipdroid比较轻量级，着重基于java开发（音频codec除外），由于其音视频编码以及P2P传输这一块略显不足，不太好做定制化开发和优化。
imsdroid,遗憾就是直接调用webrtc的库，而最近webrtc更新的比较频繁，开发比较活跃。如果要自己在imsdroid上更新webrtc担心兼容性问题，
希望imsdroid可以直接把需要的webrtc相关源码包进去。csipsimple的话，都是围绕pjsip的，webrtc等都是以pjsip插件形式扩充的,
类似gstreamer. webrtc如果有技术实力的开发公司个人还是觉得可以选择这个来做，一个是google的原因，一个是其视频通话相关关键技术都比较成熟的原因。
个人觉得如果能做出来，效果会不错的。


