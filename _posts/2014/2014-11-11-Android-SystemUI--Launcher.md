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



Status Bar 显示

	super.start(); // calls createAndAddWindows()
		BaseStatusBar start()
	PhoneStatusBar createAndAddWindows
		addStatusBarWindow
		makeStatusBarView
				mStatusBarWindow = (StatusBarWindowView) View.inflate(context, 
					R.layout.super_status_bar, null); 
				mStatusBarWindow.setOnTouchListener(new View.OnTouchListener() { 

			mStatusBarView = (PhoneStatusBarView) mStatusBarWindow.findViewById(R.id.status_bar);
			mStatusBarView.setBar(this);

			mNotificationPanel = 
				(NotificationPanelView) mStatusBarWindow.findViewById(R.id.notification_panel);
			mNotificationPanel.setStatusBar(this);

			mNavigationBarView = 
				(NavigationBarView) View.inflate(context, R.layout.navigation_bar, null); 
			mNavigationBarView.setOnTouchListener(new View.OnTouchListener() { 

		mWindowManager.addView(mStatusBarWindow, lp);


Status Bar 电池状态信息更新

	makeStatusBarView
		mBatteryController = new BatteryController(mContext);
		mBatteryController.addLabelView((TextView)mStatusBarView.findViewById(R.id.battery_text));

	BatteryController extends BroadcastReceiver
		onReceive
			if (action.equals(Intent.ACTION_BATTERY_CHANGED)) { 
				final int level = intent.getIntExtra(BatteryManager.EXTRA_LEVEL, 0); 
				int N = mLabelViews.size();
				for (int i=0; i<N; i++) { 
					TextView v = mLabelViews.get(i); 
					v.setText( ...

------- 电池图标的更新，花了很多时间没有找到
	BatteryMeterView


Status bar 时间信息更新

	Clock
		public void onReceive(Context context, Intent intent) { 
			if (action.equals(Intent.ACTION_TIMEZONE_CHANGED)) {  
			
			updateClock
			getSmallTime



Status Bar 事件通知
	比如：插拔SD卡，USB线，程序安装




Navigation Bar 显示，按键处理

	PhoneStatusBar start()
		addNavigationBar
			mWindowManager.addView(mNavigationBarView, getNavigationBarLayoutParams());
				// mNavigationBarView init at makeStatusBarView










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


































