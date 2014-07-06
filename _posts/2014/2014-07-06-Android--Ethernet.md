---
layout: post
title: Android Ethernet
categories:
- programmer
tags:
- android
---



һ	�漰�ļ�

	1	packages/app/Settings/:    //Setting�����ѡ�����
		packages/apps/Settings/src/com/android/settings/ethernet/EthernetSettings.java
		packages/apps/Settings/src/com/android/settings/ethernet/EthernetEnabler.java
		packages/apps/Settings/src/com/android/settings/ethernet/EthernetConfigDialog.java

	2	frameworks/base/ :
		SystemUI:   //״̬����status_bar����ʾ���ִ���
		frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/policy/NetworkController.java
		frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/SignalClusterView.java    //��ʵstatusbar

		ConnectivityService:
		frameworks/base/services/java/com/android/server/ConnectivityService.java   //������ethernet���ֳ������ʼ��

		jni:
		frameworks/base/core/jni/android_net_ethernet.cpp  //�¼ӵ�һЩjni

		�������е�ethernet��
		frameworks/base/services/java/com/android/server/EthernetService.java
		frameworks/base/services/java/com/android/server/NetworkManagementService.java
		frameworks/base/core/java/android/net/NetworkStats.java 

		����ӵ�ethernet��
		frameworks/base/ethernet/*  // ������Ҫethernet���֣�java api ���롣
		frameworks/base/ethernet/java/android/net/ethernet/EthernetManager.java


��	����android ethernet �ĳ�������

	1	netcfg
		netcfg   //�鿴ip���
		netcfg eth0 up dhcp   //ͨ��dhcp �Զ���ȡip������

	2	ifconfig
		ifconfig eth0 128.224.156.81 up

		ifconfig eth0 128.224.156.81 netmask 255.255.255.0 up

	3	gateway ����
		route add default gw 192.168.0.1 dev eth0

	4	dns ����
		echo "nameserver 128.224.160.11" > resolv.conf
		nameserver 128.224.160.11

		setprop net.dns1 128.224.160.11
		setprop net.dns2 147.11.100.30

	5	mac adddr
		ifconfig eth0 hw ether 00:11:22:33:44:55


��	�������̷���

1	ethernet �ĳ�ʼ������

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


2	�������ȼ�����


ϵͳ����ʱEthernetStateTracker�ͻᴴ��EthernetMonitor�߳�����������״̬��Ϣ�� ������ͨ��jni����c������ɵģ� ԭ����Ǽ���NetLink Socket�� 
�������ӻ�Ͽ�ʱ���socket�˿ھͻ���յ���Ϣ����������������ʱ��EthernetMonitor�߳̾ͻ��EthernetStateTracker����EVENT_HW_PHYCONNECT��Ϣ�� 
EthernetStateTracker��ȡ������Ϣ�� ����������Settings����ΪDHCP, �ͻ�ͨ��DhcpHandle��DhcpThread����EVENT_DHCP_START��Ϣ�� 
DhcpThread�ͻ�����NetUtil.runDhcp()���Զ���ȡ�����ַ��������EthernetMonitor�߳��ֻ��յ�������ͨ����Ϣ�� ��ͨ��EVENT_HW_CONNECTED֪ͨ
EthernetStateTracker EthernetStateTracker���Żᷢ��EVENT_STATE_CHANGED��Ϣ��ConnectivityService��


�ص��������ȼ��ٲõĽ׶Σ������������úú󣬻��Զ������ȼ��ٲõĵط�������
frameworks/base/services/java/com/android/server/ConnectivityService.java��NetworkStateTrackerHandler��handleMessage�С�
�����ٲþ�������case NetworkStateTracker.EVENT_STATE_CHANGED:�����֧��handleCaptivePortalTrackerCheck(info);���еģ��ٽ�һ��������isNewNetTypePreferredOverCurrentNetType�������true���л����µ����磬��֮���л���

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

ת�����������ԣ�
(�������ӵ��������� �� ����ƫ�ã� �� (�����������������ȼ� ���� ��ǰ�����������ȼ�)) 
               ���� 
              ��ƫ���������ǵ�ǰ�������ͣ���
              ���� 
�����ӵ���������     �� type                     ������wifi
ƫ��                 �� mNetworkPreference        ������wifi
�����������������ȼ� �� mNetConfigs[type].priority  wifi���ȼ� ������1
��ǰ�����������ȼ�   �� ��̫�����ȼ�             ������4
��ǰ��������         �� mActiveDefaultNetwork     ��������̫��
frameworks\base\core\res\res\values-large\config.xml ���������������ͣ�






��Ҫ����������̣�

	Ethernet
		Functional requirements
	1	Automatic detection of net cable plug in/out
	2	Network configuration
		>	automatic configure network ofter android boot up
		>	configure network according to user choice
			-	support dhcp mode
			-	support static ip mode
	3	Network status notification
	4	UI
	5	Data storage


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





























