---
layout: post
title: Android Ethernet
categories:
- programmer
tags:
- android
---



一  涉及文件

    1   packages/app/Settings/:    //Setting中添加选项代码
        packages/apps/Settings/src/com/android/settings/ethernet/EthernetSettings.java
        packages/apps/Settings/src/com/android/settings/ethernet/EthernetEnabler.java
        packages/apps/Settings/src/com/android/settings/ethernet/EthernetConfigDialog.java

    2   frameworks/base/ :
        SystemUI:   //状态栏（status_bar）显示部分代码
        frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/policy/NetworkController.java
        frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/SignalClusterView.java    //现实statusbar

        ConnectivityService:
        frameworks/base/services/java/com/android/server/ConnectivityService.java   //这里是ethernet部分程序的起始点

        jni:
        frameworks/base/core/jni/android_net_ethernet.cpp  //新加的一些jni

        本来就有的ethernet：
        frameworks/base/services/java/com/android/server/EthernetService.java
        frameworks/base/services/java/com/android/server/NetworkManagementService.java
        frameworks/base/core/java/android/net/NetworkStats.java

        新添加的ethernet：
        frameworks/base/ethernet/*  // 这是主要ethernet部分，java api 代码。
        frameworks/base/ethernet/java/android/net/ethernet/EthernetManager.java

二  调试android ethernet 的常用命令

    1   netcfg
        netcfg   //查看ip情况
        netcfg eth0 up dhcp   //通过dhcp 自动获取ip和网关

    2   ifconfig
        ifconfig eth0 128.224.156.81 up

        ifconfig eth0 128.224.156.81 netmask 255.255.255.0 up

    3   gateway 配置
        route add default gw 192.168.0.1 dev eth0

    4   dns 配置
        echo "nameserver 128.224.160.11" > resolv.conf
        nameserver 128.224.160.11

        setprop net.dns1 128.224.160.11
        setprop net.dns2 147.11.100.30

    5   mac adddr
        ifconfig eth0 hw ether 00:11:22:33:44:55


三  基本流程分析

1   ethernet 的初始化过程

    ConnectivityService.java
        getActiveNetworkInfo -> getNetworkInfo ->
        ConnectivityService
            // Load device network attributes from resources
            com.android.internal.R.array.radioAttributes
            com.android.internal.R.array.networkAttributes

            case ConnectivityManager.TYPE_ETHERNET:
                mNetTrackers[netType] = EthernetDataTracker.getInstance();
                mNetTrackers[netType].startMonitoring(context, mHandler);
                break;

2   网络优先级管理


系统启动时EthernetStateTracker就会创建EthernetMonitor线程来监听网络状态信息， 监听的通过jni调用c代码完成的， 原理就是监听NetLink Socket，
网络连接或断开时这个socket端口就会接收到信息。当有网线连接上时，EthernetMonitor线程就会给EthernetStateTracker发送EVENT_HW_PHYCONNECT消息，
EthernetStateTracker读取配置信息， 由于我们在Settings设置为DHCP, 就会通过DhcpHandle给DhcpThread发送EVENT_DHCP_START信息，
DhcpThread就会运行NetUtil.runDhcp()来自动获取网络地址。接下来EthernetMonitor线程又会收到网络已通的消息， 并通过EVENT_HW_CONNECTED通知
EthernetStateTracker EthernetStateTracker接着会发送EVENT_STATE_CHANGED消息给ConnectivityService。


重点在于优先级仲裁的阶段，各自网络配置好后，会自动到优先级仲裁的地方报道：
frameworks/base/services/java/com/android/server/ConnectivityService.java的NetworkStateTrackerHandler的handleMessage中。
分析仲裁具体是在case NetworkStateTracker.EVENT_STATE_CHANGED:这个分支的handleCaptivePortalTrackerCheck(info);进行的，再进一步是这里isNewNetTypePreferredOverCurrentNetType如果返回true就切换到新的网络，反之不切换。

    private boolean isNewNetTypePreferredOverCurrentNetType(int type) {
        if (DBG) {
            log("type = " + type);
            log("mNetworkPreference = " + mNetworkPreference);
            log("mNetConfigs[mActiveDefaultNetwork].priority = " + mNetConfigs[mActiveDefaultNetwork].priority);
            log(" mNetConfigs[type].priority = "+ mNetConfigs[type].priority);
            log("mNetworkPreference = " + mNetworkPreference);
            log("mActiveDefaultNetwork = " + mActiveDefaultNetwork);
            log("mActiveDefaultNetwork =" + mActiveDefaultNetwork);
        }
        if ((type != mNetworkPreference &&
                    mNetConfigs[mActiveDefaultNetwork].priority >
                    mNetConfigs[type].priority) ||
                mNetworkPreference == mActiveDefaultNetwork) return false;
        return true;
    }

转换成人类语言：
(（新连接的网络类型 不 等于偏好） 且 (新连接网络类型优先级 低于 当前网络类型优先级))
               或者
              （偏好类型正是当前网络类型））
              则不切
新连接的网络类型     ＝ type                     这里是wifi
偏好                 ＝ mNetworkPreference        这里是wifi
新连接网络类型优先级 ＝ mNetConfigs[type].priority  wifi优先级 这里是1
当前网络类型优先级   ＝ 以太网优先级             这里是4
当前网络类型         ＝ mActiveDefaultNetwork     这里是以太网
frameworks\base\core\res\res\values-large\config.xml 中有这种网络类型：






需要理解整个流程：

    Ethernet
        Functional requirements
    1   Automatic detection of net cable plug in/out
    2   Network configuration
        >   automatic configure network ofter android boot up
        >   configure network according to user choice
            -   support dhcp mode
            -   support static ip mode
    3   Network status notification
    4   UI
    5   Data storage


    get network info from kernel
        get all the net devices from kernel, store then in interface_info_t
        create a AF_NLINK socket to receive net devices status change
        get net device count
        get net device name with the index argument

    implement JNI
        java -> JNI ->

    monitor net device status
        we can detect the net calbe plug in/out

    handler net device status change
        we can notify user about network status
        we can support dhcp and static ip

    start a service

    UI

    data storage

