---
layout: post
title: Android--Thread-Handler-Looper-Message
categories:
- programmer
tags:
- android_basis
---


##Thread


##Handler
主要处理Message的类，负责将Message添加到消息队列，以及对消息队列中的消息进行处理。一般Handler的实例都是与一个线程和
该线程的消息队列一起使用，一旦创建了一个新的Handler实例，系统就把该实例与一个线程和该线程的消息队列捆绑起来，这样就可以
发送消息和runnable对象给该消息队列，并在消息队列出口处理它们。
两个主要作用：1)安排消息或Runnable在某个线程中某个时间段执行。2)安排一个动作在另外一个线程执行。


##Looper
负责管理线程的消息队列和消息循环，具体实现请参考Looper的源码。 
可以通过Loop.myLooper()得到当前线程的Looper对象，
通过Loop.getMainLooper()可以获得当前进程的主线程的Looper对象。

Looper的主要作用就是循环迭代MessageQueue，默认情况下没有跟线程相关联的消息循环，必须在线程中调用prepare()方法，
运行循环。但是在我们的Activity的Main()函数中，系统默认调用了
所以我们可以通过getMainLooper()获得一个MainLooper。


##Message
消息，就是线程间通信的数据单元，里面可以封装各种信息，Android系统中消息是通过Message类来封装的。
Message这个类实现了Parcelable接口，所以可以通过Intent或IPC传递。定义了一个能够发送给Handler对象的消息，
包含了消息的描述和任意数据对象。主要包含两个int类型字段和一个Object类型字段。我们可以通过Message的构造函数生成一个消息，
但是一般是通过调用Message.obtain()方法或Handler.obtainMessage()方法来构造一个消息。


通过下图可以清晰显示出UI Thread, Worker Thread, Handler, Massage Queue, Looper之间的关系：
![Alt text](/media/images/2014/Anddroid--Thread-Handler-Looper-Message_01.png "Anddroid--Thread-Handler-Looper-Message_01.png")



Android系统的消息队列和消息循环都是针对具体线程的，一个线程可以存在（当然也可以不存在）一个消息队列和一个消息循环（Looper），
特定线程的消息只能分发给本线程，不能进行跨线程，跨进程通讯。但是创建的工作线程默认是没有消息循环和消息队列的，
如果想让该线程具有消息队列和消息循环，需要在线程中首先调用Looper.prepare()来创建消息队列，然后调用Looper.loop()进入消息循环。如下例所示

	class LooperThread extends Thread {
		public Handler mHandler;
			public void run() {
				Looper.prepare();
				mHandler = new Handler() {
					public void handleMessage(Message msg) {
						// process incoming messages here
					}
				};
			Looper.loop();
		}
	}

这样你的线程就具有了消息处理机制了，在Handler中进行消息处理。


Activity是一个UI线程，运行于主线程中，Android系统在启动的时候会为Activity创建一个消息队列和消息循环（Looper）。
详细实现请参考ActivityThread.java文件。

Handler的作用是把消息加入特定的（Looper）消息队列中，并分发和处理该消息队列中的消息。构造Handler的时候可以指定一个Looper对象，
如果不指定则利用当前线程的Looper创建。详细实现请参考Looper的源码。


一个Activity中可以创建多个工作线程或者其他的组件，如果这些线程或者组件把他们的消息放入Activity的主线程消息队列，
那么该消息就会在主线程中处理了。因为主线程一般负责界面的更新操作，并且Android系统中的weget不是线程安全的，
所以这种方式可以很好的实现Android界面更新。在Android系统中这种方式有着广泛的运用。

那么另外一个线程怎样把消息放入主线程的消息队列呢？答案是通过Handle对象，只要Handler对象以主线程的Looper创建，
那么调用Handler的sendMessage等接口，将会把消息放入队列都将是放入主线程的消息队列。并且将会在Handler主线程中调用
该handler的handleMessage接口来处理消息。
	
	package com.android.demo;

	import android.app.Activity;
	import android.os.Bundle;
	import android.os.Handler;
	import android.os.Message;
	import android.util.Log;

	public class MyhandlerActivity extends Activity {
		/** Called when the activity is first created. */

		static final String TAG = "Handler";
		static final int HANDLER_TEST = 1;

		Handler h = new Handler(){
			public void handleMessage(Message msg){
				switch(msg.what){
				case HANDLER_TEST:
					Log.v(TAG, "The handler thread id ："+Thread.currentThread().getId());
				}
			}
		};

		@Override
		public void onCreate(Bundle savedInstanceState) {
			super.onCreate(savedInstanceState);
			Log.v(TAG, "The main thread id = " + Thread.currentThread().getId());

			setContentView(R.layout.main);
			new Mythread().start();
		}

		class Mythread extends Thread{
			public void run(){
				Message msg = new Message();
				msg.what = HANDLER_TEST;
				h.sendMessage(msg);
				Log.v(TAG, "The worker thread id = " + Thread.currentThread().getId() + "n");
			}
		}
	}


通过上面的分析，我们可以得出如下结论：
1	如果通过工作线程刷新界面，推荐使用handler对象来实现。
2	注意工作线程和主线程之间的竞争关系。推荐handler对象在主线程中构造完成（并且启动工作线程之后不要再修改之，否则会出现数据不一致），
	然后在工作线程中可以放心的调用发送消息SendMessage等接口。
3	除了2所述的hanlder对象之外的任何主线程的成员变量如果在工作线程中调用，仔细考虑线程同步问题。如果有必要需要加入同步对象保护该变量。
4	handler对象的handleMessage接口将会在主线程中调用。在这个函数可以放心的调用主线程中任何变量和函数，进而完成更新UI的任务。
5	Android很多API也利用Handler这种线程特性，作为一种回调函数的变种，来通知调用者。这样Android框架就可以在其线程中将消息发送到
	调用者的线程消息队列之中，不用担心线程同步的问题。





