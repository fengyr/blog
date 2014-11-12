---
layout: post
title: Android SystemUI and Launcher
categories:
- programmer
tags:
- android
---


Android 4.4.2


## SystemUI.apk

启动流程，UI加载过程

	frameworks/base/services/java/com/android/server/SystemServer.java
	initAndLoop（）
		startSystemUi(Context context)

		intent.setComponent(new ComponentName("com.android.systemui",
			"com.android.systemui.SystemUIService"));
		context.startServiceAsUser(intent, UserHandle.OWNER);

	
	frameworks/base/packages/SystemUI/src/com/android/systemui/SystemUIService.java
	private final Class<?>[] SERVICES = new Class[] {
		com.android.systemui.recent.Recents.class,
		com.android.systemui.statusbar.SystemBars.class,
		com.android.systemui.usb.StorageNotification.class,
		com.android.systemui.power.PowerUI.class,          
		com.android.systemui.media.RingtonePlayer.class,   
		com.android.systemui.settings.SettingsUI.class,    
	};


	final int N = SERVICES.length;
	for (int i=0; i<N; i++) {
		Class<?> cl = SERVICES[i];
		try {
			mServices[i] = (SystemUI)cl.newInstance(); 

			mServices[i].start();

	SystemBars start()
        mServiceMonitor = new ServiceMonitor(TAG, DEBUG,
                mContext, Settings.Secure.BAR_SERVICE_COMPONENT, this);
        mServiceMonitor.start();

	ServiceMonitor start()
		mHandler.sendEmptyMessage(MSG_START_SERVICE);
	startService()
	. . .

	SystemBars
		onNoService()
		createStatusBarFromConfig
			final String clsName = mContext.getString(R.string.config_statusBarComponent);
				// com.android.systemui.statusbar.phone.PhoneStatusBar
			cls = mContext.getClassLoader().loadClass(clsName);
			mStatusBar = (BaseStatusBar) cls.newInstance();
			mStatusBar.start();

	PhoneStatusBar start()
		super.start(); // calls createAndAddWindows()
			addStatusBarWindow
				makeStatusBarView
		addNavigationBar



Status Bar 显示，状态更新，事情通知

	super.start(); // calls createAndAddWindows()
		makeStatusBarView
			mStatusBarWindow = (StatusBarWindowView) View.inflate(context,
				R.layout.super_status_bar, null); 

			mStatusBarView = (PhoneStatusBarView) mStatusBarWindow.findViewById(R.id.status_bar);




		mWindowManager.addView(mStatusBarWindow, lp);





Navigation Bar 显示，按键处理

	addNavigationBar











## Launcher.apk

启动流程

	SystemServer.java
	ActivityManagerService.main

	systemReady
		mStackSupervisor.resumeTopActivitiesLocked();
	
		startHomeActivityLocked
		mStackSupervisor.startHomeActivity(intent, aInfo);


APPS 图标显示


WIDGETS 图标显示


点击，长按，拖动事件处理




##参考资料


































