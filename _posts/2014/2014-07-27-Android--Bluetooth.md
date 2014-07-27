---
layout: post
title: Android Bluetooth
categories:
- programmer
tags:
- android
---

## 基本介绍

以 android4.2 代码为基础

涉及代码路径

    android
    packages/apps/Settings/src/com/android/settings/bluetooth
    packages/apps/Bluetooth/src/com/android/bluetooth
    frameworks/base/core/java/android/bluetooth
    system/bluetooth
    external/bluetooth
    external/bluetooth_mtk
    device/softwinner/wing-s780bt/bluetooth
    device/softwinner/common/hardware/realtek/bluetooth

    hardware/libhardware/include/hardware/


    lichee
    linux-3.4/drivers/bluetooth
    linux-3.4/net/bluetooth
    linux-3.4/include/net/bluetooth



## 框架分析

android4.2 基本框架


   -----------------------                                      HAL
   |       Apps          |                     hardware/libhardware/include/hardware/
   |                     |       ----------------------------	------------------------------------
   | android.bluetooth   |   --- | Bluetooth HAL Interfaces |	| Bluetooth Profile HAL Interfaces |---
   -----------------------   |   ----------------------------   ------------------------------------  |
            |                |                                                                        |
          Binder            JNI  ----------------------------   ------------------------------------  |
            |                |   |    Vendor Extersions     |   |        Bluetooth Stack           |  |
   -----------------------   |   |--------------------------|   |-----------------------------------  |
   | Blueooth Servicce   |   |   | Custom Configurations    |   |      Bluetooth App Layer         |  |
   |                     |----   |                          |---|                                  |---
   | Bluetooth Profiles  |       | Custom Extersions        |   |     Bluetooth Embedded System    |
   -----------------------       ----------------------------   ------------------------------------


android提供BlueDroid作为默认的协议栈，BlueDroid分为两个部分：		
1、Bluetooth Embedded System（BTE），它实现了BT的核心功能。		
2、Bluetooth Application Layer (BTA)，用于和android framework层交互		

BT 系统服务通过JNI与BT stack交互，并且通过Binder IPC通信与应用交互。这个系统服务同时也提供给RD获取不同的BT profiles；、




## 主要流程分析

一、application Framework		
这个层的代码主要是利用android.bluetooth APIS 和 bluetooth hardware进行交互。 也就是通过Binder IPC机制调用bluetooth 进程；
代码位于framework/base/core/java/android.bluetooth/下。

以 A2DP 的连接为例

	frameworks/base/core/java/android/bluetooth/BluetoothA2dp.java
		public boolean connect(BluetoothDevice device) {
			if (DBG) log("connect(" + device + ")");
			if (mService != null && isEnabled() &&
				isValidDevice(device)) {
				try {
				    return mService.connect(device);
				} catch (RemoteException e) {
				    Log.e(TAG, "Stack:" + Log.getStackTraceString(new Throwable()));
				    return false;
				}
			}
			if (mService == null) Log.w(TAG, "Proxy not attached to service");
			return false;
		}

	private ServiceConnection mConnection = new ServiceConnection() {
		public void onServiceConnected(ComponentName className, IBinder service) {
		    if (DBG) Log.d(TAG, "Proxy object connected");
		    mService = IBluetoothA2dp.Stub.asInterface(service);


通过Binder IPC 通信机制，调用到		
packages/apps/Bluetooth/src/com/android/bluetooth/a2dp/A2dpService.java		
下一个内部私有类		

	private static class BluetoothA2dpBinder extends IBluetoothA2dp.Stub

		public boolean connect(BluetoothDevice device) {
			A2dpService service = getService();
			if (service == null) return false;
			return service.connect(device);
		}


然后调用到A2dpService的connect(BluetoothDevice)方法。		

	public boolean connect(BluetoothDevice device) {
		enforceCallingOrSelfPermission(BLUETOOTH_ADMIN_PERM,
		                               "Need BLUETOOTH ADMIN permission");

		if (getPriority(device) == BluetoothProfile.PRIORITY_OFF) {
		    return false;
		}

		int connectionState = mStateMachine.getConnectionState(device);
		if (connectionState == BluetoothProfile.STATE_CONNECTED ||
		    connectionState == BluetoothProfile.STATE_CONNECTING) {
		    return false;
		}

		mStateMachine.sendMessage(A2dpStateMachine.CONNECT, device);
		return true;
	}


	public class A2dpService extends ProfileService {
		private static final boolean DBG = false;
		private static final String TAG="A2dpService";

		private A2dpStateMachine mStateMachine;


这个过程就是Bluetooth Application Framework与Bluetooth Process的调用过程。



二、Bluetooth System Service		
Bluetooth System service位于packages/apps/Bluetooth下，它打包成一个android app包，并且在android framework 层实现
BT service 和各种profile。BT app会通过JNI调用到HAL层。

A2dpService的connect方法会发送一个StateMachine.sendMessage(A2dpStateMachine.CONNECT, device)的message，
这个message会被A2dpStateMachine对象的processMessage(Message)方法接收到：


	frameworks/base/core/java/com/android/internal/util/StateMachine.java
		public final void sendMessage(int what, Object obj) {
			// mSmHandler can be null if the state machine has quit.
			if (mSmHandler == null) return;

			mSmHandler.sendMessage(obtainMessage(what,obj));
		}

	packages/apps/Bluetooth/src/com/android/bluetooth/a2dp/A2dpStateMachine.java
		final class A2dpStateMachine extends StateMachine {

		@Override
		public boolean processMessage(Message message) {
			log("Disconnected process message: " + message.what);
			if (mCurrentDevice != null || mTargetDevice != null  || mIncomingDevice != null) {
				Log.e(TAG, "ERROR: current, target, or mIncomingDevice not null in Disconnected");
				return NOT_HANDLED;
			}

			boolean retValue = HANDLED;
			switch(message.what) {
				case CONNECT:
				    BluetoothDevice device = (BluetoothDevice) message.obj;
				    broadcastConnectionState(device, BluetoothProfile.STATE_CONNECTING,
				                   BluetoothProfile.STATE_DISCONNECTED);

				    // connectA2dpNative
				    if (!connectA2dpNative(getByteAddress(device)) ) {
				        broadcastConnectionState(device, BluetoothProfile.STATE_DISCONNECTED,
				                       BluetoothProfile.STATE_CONNECTING);
				        break;
				    }

				    synchronized (A2dpStateMachine.this) {
				        mTargetDevice = device;
				        transitionTo(mPending);
				    }
				    // TODO(BT) remove CONNECT_TIMEOUT when the stack
				    //          sends back events consistently
				    sendMessageDelayed(CONNECT_TIMEOUT, 30000);
				    break;

最重要的一句：connectA2dpNative(getByteAddress(device)；		
即会通过JNI调用到Native；		
private native boolean connectA2dpNative(byte[] address);		


三、JNI		
与android.bluetooth有关的JNI代码位于packages/apps/bluetooth/jni下，JNI 的代码会调用到HAL层，并且在确信一些BT操作被触发时，
会从HAL获取一些回调。比如当BT设备被发现时。

再回到A2dp连接的例子中来，BT System Service通过JNI会调用到com_android_bluetooth_a2dp.cpp中

	packages/apps/Bluetooth/jni/com_android_bluetooth_a2dp.cpp
		static JNINativeMethod sMethods[] = {
			{"classInitNative", "()V", (void *) classInitNative},
			{"initNative", "()V", (void *) initNative},
			{"cleanupNative", "()V", (void *) cleanupNative},
			{"connectA2dpNative", "([B)Z", (void *) connectA2dpNative},
			{"disconnectA2dpNative", "([B)Z", (void *) disconnectA2dpNative},
		};


	static jboolean connectA2dpNative(JNIEnv *env, jobject object, jbyteArray address) {
		jbyte *addr;
		bt_bdaddr_t * btAddr;
		bt_status_t status;

		ALOGI("%s: sBluetoothA2dpInterface: %p", __FUNCTION__, sBluetoothA2dpInterface);
		if (!sBluetoothA2dpInterface) return JNI_FALSE;

		addr = env->GetByteArrayElements(address, NULL);
		btAddr = (bt_bdaddr_t *) addr;
		if (!addr) {
		    jniThrowIOException(env, EINVAL);
		    return JNI_FALSE;
		}

		// connect
		if ((status = sBluetoothA2dpInterface->connect((bt_bdaddr_t *)addr)) != BT_STATUS_SUCCESS) {
		    ALOGE("Failed HF connection, status: %d", status);
		}
		env->ReleaseByteArrayElements(address, addr, 0);
		return (status == BT_STATUS_SUCCESS) ? JNI_TRUE : JNI_FALSE;
	}


重点代码是：status = sBluetoothA2dpInterface->connect((bt_bdaddr_t *)addr)；		
这个sBluetoothA2dpInterface结构体对象是在initNative(JNIEnv *env, jobject object)方法时得到的。		

	static const btav_interface_t *sBluetoothA2dpInterface = NULL;

	static void initNative(JNIEnv *env, jobject object) {
		const bt_interface_t* btInf;

		if ( (btInf = getBluetoothInterface()) == NULL) {
			ALOGE("Bluetooth module is not loaded");
			return;
		}


		if ( (sBluetoothA2dpInterface = (btav_interface_t *)
			  btInf->get_profile_interface(BT_PROFILE_ADVANCED_AUDIO_ID)) == NULL) {
			ALOGE("Failed to get Bluetooth A2DP Interface");
			return;
		}

	packages/apps/Bluetooth/jni/com_android_bluetooth_btservice_AdapterService.cpp
		const bt_interface_t* getBluetoothInterface() {
			return sBluetoothInterface;
		}

		static void classInitNative(JNIEnv* env, jclass clazz) {
			char value[PROPERTY_VALUE_MAX];
			property_get("bluetooth.mock_stack", value, "");

			const char *id = (strcmp(value, "1")? BT_STACK_MODULE_ID : BT_STACK_TEST_MODULE_ID);

			err = hw_get_module(id, (hw_module_t const**)&module);

			if (err == 0) {
				hw_device_t* abstraction;
				err = module->methods->open(module, id, &abstraction);
				if (err == 0) {
				    bluetooth_module_t* btStack = (bluetooth_module_t *)abstraction;
				    sBluetoothInterface = btStack->get_bluetooth_interface();
				} else {
				   ALOGE("Error while opening Bluetooth library");
				}
			} else {
				ALOGE("No Bluetooth Library found");
			}



四、HAL		
硬件抽象层定义android.bluetooth APIs和BT process调用的标准接口，并且你必须实现这些接口来让你的BT hardware功能运行正常。
BT HAL的的头文件位于hardware/libhardware/include/hardware/bluetooth.h和hardware/libhardware/include/hardware/bt_*.h
文件中。

JNI中sBluetoothA2dpInterface是一个btav_interface_t结构体，位于hardware/libhardware/include/hardware/bt_av.h中，
定义为：

	hardware/libhardware/include/hardware/bluetooth.h
		typedef struct {
			/** set to sizeof(bt_interface_t) */
			size_t size;

			const void* (*get_profile_interface) (const char *profile_id);

			/** Bluetooth Test Mode APIs - Bluetooth must be enabled for these APIs */
			/* Configure DUT Mode - Use this mode to enter/exit DUT mode */
			int (*dut_mode_configure)(uint8_t enable);

			/* Send any test HCI (vendor-specific) command to the controller. Must be in DUT Mode */
			int (*dut_mode_send)(uint16_t opcode, uint8_t *buf, uint8_t len);
		} bt_interface_t;


	hardware/libhardware/include/hardware/bt_av.h
		typedef struct {

			/** set to sizeof(btav_interface_t) */
			size_t          size;
			/**
			 * Register the BtAv callbacks
			 */
			bt_status_t (*init)( btav_callbacks_t* callbacks );

			/** connect to headset */
			bt_status_t (*connect)( bt_bdaddr_t *bd_addr );

			/** dis-connect from headset */
			bt_status_t (*disconnect)( bt_bdaddr_t *bd_addr );

			/** Closes the interface. */
			void  (*cleanup)( void );
		} btav_interface_t;




五、BT stack		
作为默认的BT stack，代码位于external/bluetooth/bluedroid下，这个stack实现了通用的BT HAL并且也可以通过扩展和改变配置来自定义。
A2dp的连接会调用到external/bluetooth/bluedroid/btif/src/btif_av.c的connect方法。

	external/bluetooth/bluedroid/btif/src/btif_av.c
		static bt_status_t connect(bt_bdaddr_t *bd_addr)
		｛
			BTIF_TRACE_EVENT1("%s", __FUNCTION__);
			CHECK_BTAV_INIT();

			return btif_queue_connect(UUID_SERVCLASS_AUDIO_SOURCE, bd_addr, connect_int);
		｝

	external/bluetooth/bluedroid/btif/src/btif_profile_queue.c
		bt_status_t btif_queue_connect(uint16_t uuid, const bt_bdaddr_t *bda,
				                btif_connect_cb_t *connect_cb)
		{
			connect_node_t node;
			memset(&node, 0, sizeof(connect_node_t));
			memcpy(&(node.bda), bda, sizeof(bt_bdaddr_t));
			node.uuid = uuid;
			node.p_cb = connect_cb;

			return btif_transfer_context(queue_int_handle_evt, BTIF_QUEUE_CONNECT_EVT,
				                  (char*)&node, sizeof(connect_node_t), NULL);
		}

	external/bluetooth/bluedroid/btif/src/btif_core.c
		bt_status_t btif_transfer_context (tBTIF_CBACK *p_cback, UINT16 event, char* p_params, int param_len,
			tBTIF_COPY_CBACK *p_copy_cback)
		{
			tBTIF_CONTEXT_SWITCH_CBACK *p_msg;

			btif_sendmsg(p_msg);


		void btif_sendmsg(void *p_msg)
		{
			GKI_send_msg(BTIF_TASK, BTU_BTIF_MBOX, p_msg);
		}


	external/bluetooth/bluedroid/gki/common/gki_buffer.c
		void GKI_send_msg (UINT8 task_id, UINT8 mbox, void *msg)



六、Vendor extension		
为了追踪添加自定义拓展和一个HCI层，你可以创建一个libbt-vendor模块并且指定这些组件



## 附一 参考资料
1	android4.2 Bluetooth记录01-结构和代码分布		
	http://my.eoe.cn/cnhua5/archive/10607.html		
2	Android 4.2蓝牙介绍		
	http://blog.csdn.net/innost/article/details/9187199		
3	07.Android之Bluetooth		
	http://blog.csdn.net/jjunjoe/article/details/6928913		

