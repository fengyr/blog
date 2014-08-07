---
layout: post
title: Android Vold & Mount
categories:
- programmer
tags:
- android
---



## Vold 启动

	VolumeManager
	NetlinkManager
	CommandListener
	FrameworkListener
	NetlinkHandler
	NetlinkListener


vold启动在init.rc中：

	service vold /system/bin/vold
    socket vold stream 0660 root mount
    ioprio be 2

注意这里创建了一个socket，用于vold和FrameWork层通信


vold 入口main函数：

	int main() {
		VolumeManager *vm;  
		CommandListener *cl;  
		NetlinkManager *nm;  
	  
		SLOGI("Vold 2.1 (the revenge) firing up");  
	  
		mkdir("/dev/block/vold", 0755);  

这里建立了一个/dev/block/vold目录用于放置后面建立的vold节点


	/* Create our singleton managers */
	if (!(vm = VolumeManager::Instance())) {
	   SLOGE("Unable to create VolumeManager");
	   exit(1);
	};

	if (!(nm = NetlinkManager::Instance())) {
	   SLOGE("Unable to create NetlinkManager");
	   exit(1);
	};

这里创建了VolumeManager和NetlinkManager两个实例，
VolumeManager主要负责Voluem的一些管理，NetlinkManager主要负责管理与内核之间的通信


	cl = new CommandListener();

这里首先创建了CommandListener,CommandListener主要负责与FrameWork层的通信，
处理从FrameWork层收到的各种命令


	CommandListener::CommandListener() :
		             FrameworkListener("vold") {
		registerCmd(new DumpCmd());
		registerCmd(new VolumeCmd());
		registerCmd(new AsecCmd());
		registerCmd(new ObbCmd());
		registerCmd(new ShareCmd());
		registerCmd(new StorageCmd());
		registerCmd(new XwarpCmd());
	}

这里注册了各种命令，注意FrameworkListener("vold")，FrameworkListener又继承了SocketListener，
最终"vold"传到了SocketListener里面。


	vm->setBroadcaster((SocketListener *) cl);
	nm->setBroadcaster((SocketListener *) cl);

设置了Broadcaster，后面给FrameWork层发送消息就跟它有关了


	if (process_config(vm)) {
		SLOGE("Error reading configuration (%s)... continuing anyways", strerror(errno));  
	}

解析vold.fstab

解析了vold.fstab之后 ，就开始启动NetlinkManager

	if (nm->start()) {
		SLOGE("Unable to start NetlinkManager (%s)", strerror(errno));
		exit(1);
	}

我们跟进去看看：

	int NetlinkManager::start() {
		struct sockaddr_nl nladdr;  
		int sz = 64 * 1024;  
		int on = 1;
	  
		memset(&nladdr, 0, sizeof(nladdr));  
		nladdr.nl_family = AF_NETLINK;  
		nladdr.nl_pid = getpid();  
		nladdr.nl_groups = 0xffffffff;  
	  
		if ((mSock = socket(PF_NETLINK,  
				SOCK_DGRAM,NETLINK_KOBJECT_UEVENT)) < 0) {//注册UEVENT事件，用于接收内核消息
		    SLOGE("Unable to create uevent socket: %s", strerror(errno));
		    return -1;
		}  
	  
		if (setsockopt(mSock, SOL_SOCKET, SO_RCVBUFFORCE, &sz, sizeof(sz)) < 0) {  
		    SLOGE("Unable to set uevent socket options: %s", strerror(errno));
		    return -1;
		}  
	  
		if (setsockopt(mSock, SOL_SOCKET, SO_REUSEADDR,  &on, sizeof(on)) < 0) {  
		    LOGE("Unable to set SO_REUSEADDR options: %s", strerror(errno));
		    return -1;
		}  
		  
		if (bind(mSock, (struct sockaddr *) &nladdr, sizeof(nladdr)) < 0) {  
		    SLOGE("Unable to bind uevent socket: %s", strerror(errno));
		    return -1;
		}  
	  
		mHandler = new NetlinkHandler(mSock);//NetlinkHandler用于对接收到的内核消息进行处理  
		if (mHandler->start()) {//开始监听内核消息  
		    SLOGE("Unable to start NetlinkHandler: %s", strerror(errno));
		    return -1;
		}  
		return 0;  
	}



我们跟进mHandler->start()最终调用 SocketListener::startListener()

	int SocketListener::startListener() {
	  
		if (!mSocketName && mSock == -1) { //这里mSock 刚赋值了
		    SLOGE("Failed to start unbound listener");
		    errno = EINVAL;
		    return -1;
		} else if (mSocketName) {
		    if ((mSock = android_get_control_socket(mSocketName)) < 0) {
		        SLOGE("Obtaining file descriptor socket '%s' failed: %s",
		             mSocketName, strerror(errno));
		        return -1;
		    }
		}
	  
		if (mListen && listen(mSock, 4) < 0) {
		    SLOGE("Unable to listen on socket (%s)", strerror(errno));
		    return -1;
		} else if (!mListen)
		    mClients->push_back(new SocketClient(mSock));
	  
		if (pipe(mCtrlPipe)) {//建立管道，用于后面的关闭监听循环
		    SLOGE("pipe failed (%s)", strerror(errno));
		    return -1;
		}
	  
		if (pthread_create(&mThread, NULL, SocketListener::threadStart, this)) {
		    SLOGE("pthread_create (%s)", strerror(errno));
		    return -1;
		}
	  
		return 0;
	}




我们再看下threadStart，最终调用runListener

	void SocketListener::runListener() {
	  
		while(1) {  
		    SocketClientCollection::iterator it;
		    fd_set read_fds;
		    int rc = 0;
		    int max = 0;
	  
		    FD_ZERO(&read_fds);
	  
		    if (mListen) {
		        max = mSock;
		        FD_SET(mSock, &read_fds);
		    }
	  
		    FD_SET(mCtrlPipe[0], &read_fds);  //把mCtrlPipe[0]也加入监听中
		    if (mCtrlPipe[0] > max)
		        max = mCtrlPipe[0];
	  
		    pthread_mutex_lock(&mClientsLock);
		    for (it = mClients->begin(); it != mClients->end(); ++it) {
		        FD_SET((*it)->getSocket(), &read_fds);
		        if ((*it)->getSocket() > max)
		            max = (*it)->getSocket();
		    }
		    pthread_mutex_unlock(&mClientsLock);
	  
		    if ((rc = select(max + 1, &read_fds, NULL, NULL, NULL)) < 0) {//阻塞直到有数据到来
		        SLOGE("select failed (%s)", strerror(errno));
		        sleep(1);
		        continue;
		    } else if (!rc)
		        continue;
	  
			//mCtrlPipe[0]有数据，则结束循环，注意是在stopListener的时候 往mCtrlPipe[1]写数据 
		    if (FD_ISSET(mCtrlPipe[0], &read_fds))
		        break; 
		    if (mListen && FD_ISSET(mSock, &read_fds)) {
		        struct sockaddr addr;
		        socklen_t alen = sizeof(addr);
		        int c;
	  
		        if ((c = accept(mSock, &addr, &alen)) < 0) {//有新的连接来了，主要是FrameWork层的，
		            SLOGE("accept failed (%s)", strerror(errno));
		            sleep(1);
		            continue;
		        }
		        pthread_mutex_lock(&mClientsLock);
		        mClients->push_back(new SocketClient(c));//加到监听的列表
		        pthread_mutex_unlock(&mClientsLock);
		    }
	  
		    do {
		        pthread_mutex_lock(&mClientsLock);
		        for (it = mClients->begin(); it != mClients->end(); ++it) {
		            int fd = (*it)->getSocket();
		            if (FD_ISSET(fd, &read_fds)) {
		                pthread_mutex_unlock(&mClientsLock);
		                if (!onDataAvailable(*it)) {//处理消息
		                    close(fd);
		                    pthread_mutex_lock(&mClientsLock);
		                    delete *it;
		                    it = mClients->erase(it);
		                    pthread_mutex_unlock(&mClientsLock);
		                }
		                FD_CLR(fd, &read_fds);
		                pthread_mutex_lock(&mClientsLock);
		                continue;
		            }
		        }
		        pthread_mutex_unlock(&mClientsLock);
		    } while (0);
		}  
	}

这样，就开始了监听来自内核的事件


	coldboot("/sys/block");
	coldboot("/sys/class/switch");
	  
	/* 
	 * Now that we're up, we can respond to commands 
	 */
	if (cl->startListener()) {
		SLOGE("Unable to start CommandListener (%s)", strerror(errno));  
		exit(1);  
	}

这里主要看cl->startListener,也跟前面 的一样，调用SocketListener::startListener()，注意这时

	if (!mSocketName && mSock == -1) {
		SLOGE("Failed to start unbound listener");  
		errno = EINVAL;  
		return -1;  
	} else if (mSocketName) {
		if ((mSock = android_get_control_socket(mSocketName)) < 0) {  
		    SLOGE("Obtaining file descriptor socket '%s' failed: %s",
		         mSocketName, strerror(errno));
		    return -1;
		}  
	}

这里mSocketName 为"vold",mSock = -1，所以会调用android_get_control_socket，我们看下这个函数


	/* 
	 * android_get_control_socket - simple helper function to get the file 
	 * descriptor of our init-managed Unix domain socket. `name' is the name of the 
	 * socket, as given in init.rc. Returns -1 on error. 
	 * 
	 * This is inline and not in libcutils proper because we want to use this in 
	 * third-party daemons with minimal modification. 
	 */  
	static inline int android_get_control_socket(const char *name)  
	{  
		char key[64] = ANDROID_SOCKET_ENV_PREFIX;  
		const char *val;  
		int fd;  
	  
		/* build our environment variable, counting cycles like a wolf ... */  
	#if HAVE_STRLCPY  
		strlcpy(key + sizeof(ANDROID_SOCKET_ENV_PREFIX) - 1,  
		    name,  
		    sizeof(key) - sizeof(ANDROID_SOCKET_ENV_PREFIX));  
	#else   /* for the host, which may lack the almightly strncpy ... */  
		strncpy(key + sizeof(ANDROID_SOCKET_ENV_PREFIX) - 1,  
		    name,  
		    sizeof(key) - sizeof(ANDROID_SOCKET_ENV_PREFIX));  
		key[sizeof(key)-1] = '\0';  
	#endif  
	  
		val = getenv(key);  
		if (!val)  
		    return -1;  
	  
		errno = 0;  
		fd = strtol(val, NULL, 10);  
		if (errno)  
		    return -1;  
	  
		return fd;  
	}  

这里面通过组合成一个环境变量名，然后获取对应的值，那么这个值是什么时候设置的呢，
我们看下系统初始化的时候调用的service_start

	void service_start(struct service *svc, const char *dynamic_args)  
	{  
	   ...  
	   for (si = svc->sockets; si; si = si->next) {  
		        int socket_type = (  
		                !strcmp(si->type, "stream") ? SOCK_STREAM :  
		                    (!strcmp(si->type, "dgram") ? SOCK_DGRAM : SOCK_SEQPACKET));  
		        int s = create_socket(si->name, socket_type,  
		                              si->perm, si->uid, si->gid);  
		        if (s >= 0) {  
		            publish_socket(si->name, s);  
		        }  
		    }  
	} 

跟进publish_socket

	static void publish_socket(const char *name, int fd)  
	{  
		char key[64] = ANDROID_SOCKET_ENV_PREFIX;  
		char val[64];  
	  
		strlcpy(key + sizeof(ANDROID_SOCKET_ENV_PREFIX) - 1,  
		        name,  
		        sizeof(key) - sizeof(ANDROID_SOCKET_ENV_PREFIX));  
		snprintf(val, sizeof(val), "%d", fd);  
		add_environment(key, val);  
	  
		/* make sure we don't close-on-exec */  
		fcntl(fd, F_SETFD, 0);  
	} 

没错，就是这里设置了这个环境变量的值


Ok，到这里，vold基本就启动起来了，基本的通信环境也已经搭建好了，就等着u盘插入后kernel的消息的



## MountService启动

vold模块已经启动了，通信的机制也已经建立起来了，接下来我们分析一下MountService的启动，
也就是我们FrameWork层的启动

	SystemServer
	MountService
	NativeDaemonConnection
	LocalSocket
	LocalSocketAddress


MountService的启动在SystemServer.java中

	try {  
	   /* 
		* NotificationManagerService is dependant on MountService, 
		* (for media / usb notifications) so we must start MountService first. 
		*/  
	   Slog.i(TAG, "Mount Service");  
	   ServiceManager.addService("mount", new MountService(context));  
	} catch (Throwable e) {  
	   Slog.e(TAG, "Failure starting Mount Service", e);  
	}  

这里new 了一个MountService，并把service添加到了ServiceManager,我们看下MountService的构造函数：

	/** 
	 * Constructs a new MountService instance 
	 * 
	 * @param context  Binder context for this service 
	 */  
	public MountService(Context context) {  
		mContext = context;  
	  
		// XXX: This will go away soon in favor of IMountServiceObserver  
		mPms = (PackageManagerService) ServiceManager.getService("package");//获取包管理服务  
	  
		mContext.registerReceiver(mBroadcastReceiver,  
		        new IntentFilter(Intent.ACTION_BOOT_COMPLETED), null, null);//注册广播接收器  
	  
		mHandlerThread = new HandlerThread("MountService");//处理消息  
		mHandlerThread.start();  
		mHandler = new MountServiceHandler(mHandlerThread.getLooper());  
	  
		// Add OBB Action Handler to MountService thread.  
		mObbActionHandler = new ObbActionHandler(mHandlerThread.getLooper());  
	  
		/* 
		 * Vold does not run in the simulator, so pretend the connector thread 
		 * ran and did its thing. 
		 */  
		if ("simulator".equals(SystemProperties.get("ro.product.device"))) {  
		    mReady = true;  
		    mUmsEnabling = true;  
		    return;  
		}  
	  
		/* 
		 * Create the connection to vold with a maximum queue of twice the 
		 * amount of containers we'd ever expect to have. This keeps an 
		 * "asec list" from blocking a thread repeatedly. 
		 */  
		mConnector = new NativeDaemonConnector(this, "vold",  
		        PackageManagerService.MAX_CONTAINERS * 2, VOLD_TAG);  
		mReady = false;  
		Thread thread = new Thread(mConnector, VOLD_TAG);  
		thread.start();  
	}  


后面new 了一个NativeDaemonConnector，注意这里传递了一个"vold"字符串，
跟我们在vold启动的时候传给CommandListener是一样的。
NativeDaemonConnector实现了Runnable接口。接下来调用 thread.start()启动线程，我们看下它的run函数

	public void run() {    
	    while (true) {  
	        try {  
	            listenToSocket();  
	        } catch (Exception e) {  
	            Slog.e(TAG, "Error in NativeDaemonConnector", e);  
	            SystemClock.sleep(5000);  
	        }  
	    }  
	}  

在循环中调用listenToSocket函数，看下这个函数


	private void listenToSocket() throws IOException {  
		LocalSocket socket = null;  
	  
		try {  
		    socket = new LocalSocket();  
			//这里mSocket=“vold" 
		    LocalSocketAddress address = new LocalSocketAddress(mSocket,    
		            LocalSocketAddress.Namespace.RESERVED);        //注意这里的RESERVED  
	  
		    socket.connect(address);              //连接到vold模块监听的套接字处  
		    mCallbacks.onDaemonConnected();       //实现在MountService中  
	  
		    InputStream inputStream = socket.getInputStream();  
		    mOutputStream = socket.getOutputStream();  
	  
		    byte[] buffer = new byte[BUFFER_SIZE];  
		    int start = 0;  
	  
		    while (true) {  
		        int count = inputStream.read(buffer, start, BUFFER_SIZE - start); //读取消息  
		        if (count < 0) break;  
	  
		        // Add our starting point to the count and reset the start.  
		        count += start;  
		        start = 0;  
	  
		        for (int i = 0; i < count; i++) {  
		            if (buffer[i] == 0) {  
		                String event = new String(buffer, start, i - start);  
		                if (LOCAL_LOGD) Slog.d(TAG, String.format("RCV <- {%s}", event));  
	  
		                String[] tokens = event.split(" ");  
		                try {  
		                    int code = Integer.parseInt(tokens[0]);  
	  
		                    if (code >= ResponseCode.UnsolicitedInformational) {  
		                        try {  
									//实现在MountService中
		                            if (!mCallbacks.onEvent(code, event, tokens)) {  
		                                Slog.w(TAG, String.format(  
		                                        "Unhandled event (%s)", event));  
		                            }  
		                        } catch (Exception ex) {  
		                            Slog.e(TAG, String.format(  
		                                    "Error handling '%s'", event), ex);  
		                        }  
		                    }  
		                    try {  
		                        mResponseQueue.put(event);  
		                    } catch (InterruptedException ex) {  
		                        Slog.e(TAG, "Failed to put response onto queue", ex);  
		                    }  
		                } catch (NumberFormatException nfe) {  
		                    Slog.w(TAG, String.format("Bad msg (%s)", event));  
		                }  
		                start = i + 1;  
		            }  
		        }  
	  
		        // We should end at the amount we read. If not, compact then  
		        // buffer and read again.  
		        if (start != count) {  
		            final int remaining = BUFFER_SIZE - start;  
		            System.arraycopy(buffer, start, buffer, 0, remaining);  
		            start = remaining;  
		        } else {  
		            start = 0;  
		        }  
		    }  
		} catch (IOException ex) {  
		    Slog.e(TAG, "Communications error", ex);  
		    throw ex;  
		} finally {  
		    synchronized (this) {  
		        if (mOutputStream != null) {  
		            try {  
		                mOutputStream.close();  
		            } catch (IOException e) {  
		                Slog.w(TAG, "Failed closing output stream", e);  
		            }  
		            mOutputStream = null;  
		        }  
		    }  
	  
		    try {  
		        if (socket != null) {  
		            socket.close();  
		        }  
		    } catch (IOException ex) {  
		        Slog.w(TAG, "Failed closing socket", ex);  
		    }  
		}  
	}  


onDaemonConnected的实现在MountServices中，将向下下发volume list消息 获取到了磁盘的标签，
挂载点与状态，调用connect函数连接到vold模块，connetc最终调用native函数connectLocal进行连接工作，
我们看下他的jni层代码，最后调用的：


	int socket_local_client_connect(int fd, const char *name, int namespaceId,   
		    int type)  
	{  
		struct sockaddr_un addr;  
		socklen_t alen;  
		size_t namelen;  
		int err;  
	  
		err = socket_make_sockaddr_un(name, namespaceId, &addr, &alen);  
	  
		if (err < 0) {  
		    goto error;  
		}  
	  
		if(connect(fd, (struct sockaddr *) &addr, alen) < 0) {  
		    goto error;  
		}  
	  
		return fd;  
	  
	error:  
		return -1;  
	}

我们再跟进socket_make_sockaddr_un函数，这时namespaceId传的ANDROID_SOCKET_NAMESPACE_RESERVED,
所以会执行下面几句：

	case ANDROID_SOCKET_NAMESPACE_RESERVED:  
		namelen = strlen(name) + strlen(ANDROID_RESERVED_SOCKET_PREFIX);  
		/* unix_path_max appears to be missing on linux */  
		if (namelen > sizeof(*p_addr)   
			   - offsetof(struct sockaddr_un, sun_path) - 1) {  
		   goto error;  
		}  
	  
		strcpy(p_addr->sun_path, ANDROID_RESERVED_SOCKET_PREFIX);  
		//  ANDROID_RESERVED_SOCKET_PREFIX="/dev/socket/"  
		strcat(p_addr->sun_path, name);  
		break;  


注意在前面 connect  函数中的套接字的构造，使用了AF_LOCAL：


	int socket_local_client(const char *name, int namespaceId, int type)  
	{  
		int s;  
	  
		s = socket(<span style="color:#ff0000;">AF_LOCAL</span>, type, 0);  
		if(s < 0) return -1;  
	  
		if ( 0 > socket_local_client_connect(s, name, namespaceId, type)) {  
		    close(s);  
		    return -1;  
		}  
	  
		return s;  
	} 

这样，就建立了一条从FrameWork层到vold层的通信链路，后面FrameWork层就等待Vold发送消息过来了。
FrameWork层的通信也ok了，就可以等待U盘挂载了。。




## vold处理内核消息

这里要讲的是内核发信息给vold，我们在 vold启动讲到过注册了一个到内核的UEVENT事件，当有u盘插入的时候，
我们就能从这个套接字上收到内核所发出的消息了，这样就开始了vold的消息处理

	SocketListener
	NetlinkListener
	NetlinkEvent
	NetlinkHandler
	VolumeManager
	DiectVolume


在SocketListener::runListener()函数 中，我们一直在select，等待某个连接的到来或者已经的套接字上数据的到来


	if ((rc = select(max + 1, &read_fds, NULL, NULL, NULL)) < 0) {  
		SLOGE("select failed (%s)", strerror(errno));  
		sleep(1);  
		continue;  
	} else if (!rc)  
		continue;  
	  
	if (FD_ISSET(mCtrlPipe[0], &read_fds))  
		break;  
	if (mListen && FD_ISSET(mSock, &read_fds)) {  
		struct sockaddr addr;  
		socklen_t alen = sizeof(addr);  
		int c;  

		if ((c = accept(mSock, &addr, &alen)) < 0) {  
		    SLOGE("accept failed (%s)", strerror(errno));  
		    sleep(1);  
		    continue;  
		}  
		pthread_mutex_lock(&mClientsLock);  
		mClients->push_back(new SocketClient(c));  
		pthread_mutex_unlock(&mClientsLock);  
	}  

	do {  
		pthread_mutex_lock(&mClientsLock);  
		for (it = mClients->begin(); it != mClients->end(); ++it) {  
		    int fd = (*it)->getSocket();  
		    if (FD_ISSET(fd, &read_fds)) {  
		        pthread_mutex_unlock(&mClientsLock);  
		        if (!onDataAvailable(*it)) {  
		            close(fd);  
		            pthread_mutex_lock(&mClientsLock);  
		            delete *it;  
		            it = mClients->erase(it);  
		            pthread_mutex_unlock(&mClientsLock);  
		        }  
		        FD_CLR(fd, &read_fds);  
		        pthread_mutex_lock(&mClientsLock);  
		        continue;  
		    }  
		}  
		pthread_mutex_unlock(&mClientsLock);  
	} while (0);  

当某个套接字上有数据到来时，首先看这个套接字是不是listen的那个套接字，如果是则接收 连接并加到mClients链表中，
否则说明某个套接字上有数据到来，这时里是我们注册到内核的那个套接字，调用onDataAvailable函数，
这里由于多态调用的是NetlinkListener::onDataAvailable中的这个函数

	bool NetlinkListener::onDataAvailable(SocketClient *cli)  
	{  
		int socket = cli->getSocket();  
		int count;  
	  
		if ((count = recv(socket, mBuffer, sizeof(mBuffer), 0)) < 0) {  
		    SLOGE("recv failed (%s)", strerror(errno));  
		    return false;  
		}  
	  
		NetlinkEvent *evt = new NetlinkEvent();  
		if (!evt->decode(mBuffer, count)) {  
		    SLOGE("Error decoding NetlinkEvent");  
		    goto out;  
		}  
	  
		onEvent(evt);  
	out:  
		delete evt;  
		return true;  
	} 

调用recv接收数据，接着new一个NetlinkEvent并调用它的 decode函数对收到的数据进行解析：

	bool NetlinkEvent::decode(char *buffer, int size) {  
		char *s = buffer;  
		char *end;  
		int param_idx = 0;  
		int i;  
		int first = 1;  
	  
		end = s + size;  
		while (s < end) {  
		    if (first) {  
		        char *p;  
		        for (p = s; *p != '@'; p++);  
		        p++;  
		        mPath = strdup(p);  
		        first = 0;  
		    } else {  
		        if (!strncmp(s, "ACTION=", strlen("ACTION="))) {  
		            char *a = s + strlen("ACTION=");  
		            if (!strcmp(a, "add"))  
		                mAction = NlActionAdd;  
		            else if (!strcmp(a, "remove"))  
		                mAction = NlActionRemove;  
		            else if (!strcmp(a, "change"))  
		                mAction = NlActionChange;  
		        } else if (!strncmp(s, "SEQNUM=", strlen("SEQNUM=")))  
		            mSeq = atoi(s + strlen("SEQNUM="));  
		        else if (!strncmp(s, "SUBSYSTEM=", strlen("SUBSYSTEM=")))  
		            mSubsystem = strdup(s + strlen("SUBSYSTEM="));  
		        else  
		            mParams[param_idx++] = strdup(s);  
		    }  
		    s+= strlen(s) + 1;  
		}  
		return true;  
	}  

这里会对消息进行解析，解析出ACTION、DEVPATH、SUBSYSTEM等等

解析完后，就调用onEvent函数对消息进行处理，这里调用的是NetlinkHandler的onEvent函数：

	void NetlinkHandler::onEvent(NetlinkEvent *evt) {  
		VolumeManager *vm = VolumeManager::Instance();  
		const char *subsys = evt->getSubsystem();  
	  
		if (!subsys) {  
		    SLOGW("No subsystem found in netlink event");  
		    return;  
		}  
	  
		if (!strcmp(subsys, "block")) {  
		    vm->handleBlockEvent(evt);  
		} else if (!strcmp(subsys, "switch")) {  
		    vm->handleSwitchEvent(evt);  
		} else if (!strcmp(subsys, "usb_composite")) {  
		    vm->handleUsbCompositeEvent(evt);  
		} else if (!strcmp(subsys, "battery")) {  
		} else if (!strcmp(subsys, "power_supply")) {  
		}  
	}  

调用handleBlockEvent函数

	void VolumeManager::handleBlockEvent(NetlinkEvent *evt) {  
		const char *devpath = evt->findParam("DEVPATH");  
	  
		/* Lookup a volume to handle this device */  
		VolumeCollection::iterator it;  
		bool hit = false;  
		for (it = mVolumes->begin(); it != mVolumes->end(); ++it) {  
		    if (!(*it)->handleBlockEvent(evt)) {  
	#ifdef NETLINK_DEBUG  
			SLOGD("Device '%s' event handled by volume %s\n", devpath, (*it)->getLabel());  
	#endif  
		        hit = true;  
		        break;  
		    }  
		}  
	  
		if (!hit) {  
	#ifdef NETLINK_DEBUG  
		    SLOGW("No volumes handled block event for '%s'", devpath);  
	#endif  
		}  
	}  

mVolumes中我们在初始化的时候往里面add了 个DirectVolume，所以这里调用DirectVolume::handleBlockEvent

	int DirectVolume::handleBlockEvent(NetlinkEvent *evt) {  
		const char *dp = evt->findParam("DEVPATH");  
	  
		PathCollection::iterator  it;  
		for (it = mPaths->begin(); it != mPaths->end(); ++it) {  
		    if (!strncmp(dp, *it, strlen(*it))) {  
		        /* We can handle this disk */  
		        int action = evt->getAction();  
		        const char *devtype = evt->findParam("DEVTYPE");  
	  
		        if (action == NetlinkEvent::NlActionAdd) {  
		            int major = atoi(evt->findParam("MAJOR"));  
		            int minor = atoi(evt->findParam("MINOR"));  
		            char nodepath[255];  
	  
		            snprintf(nodepath,  
		                     sizeof(nodepath), "/dev/block/vold/%d:%d",  
		                     major, minor);  
		            if (createDeviceNode(nodepath, major, minor)) {  
		                SLOGE("Error making device node '%s' (%s)", nodepath,  
		                                                           strerror(errno));  
		            }  
		            if (!strcmp(devtype, "disk")) {  
		                handleDiskAdded(dp, evt);  
		            } else {  
		                handlePartitionAdded(dp, evt);  
		            }  
		        } else if (action == NetlinkEvent::NlActionRemove) {  
		            if (!strcmp(devtype, "disk")) {  
		                handleDiskRemoved(dp, evt);  
		            } else {  
		                handlePartitionRemoved(dp, evt);  
		            }  
		        } else if (action == NetlinkEvent::NlActionChange) {  
		            if (!strcmp(devtype, "disk")) {  
		                handleDiskChanged(dp, evt);  
		            } else {  
		                handlePartitionChanged(dp, evt);  
		            }  
		        } else {  
		                SLOGW("Ignoring non add/remove/change event");  
		        }  
	  
		        return 0;  
		    }  
		}  
		errno = ENODEV;  
		return -1;  
	}  

mPaths我们在parse vold.fstab把相应的解析到的路径添加进去了，我们看下这个脚本:

	ev_mount sdcard /mnt/sdcard auto /devices/platform/hiusb-ehci.0 
		/devices/platform/hi_godbox-ehci.0  

首先执行的handleDiskAdded，也就是在收到这样的消息的时候，提示有磁盘插入：

	void DirectVolume::handleDiskAdded(const char *devpath, NetlinkEvent *evt) {  
		mDiskMajor = atoi(evt->findParam("MAJOR"));  
		mDiskMinor = atoi(evt->findParam("MINOR"));  
	  
		const char *tmp = evt->findParam("NPARTS");  
		if (tmp) {  
		    mDiskNumParts = atoi(tmp);  
		} else {  
		    SLOGW("Kernel block uevent missing 'NPARTS'");  
		    mDiskNumParts = 1;  
		}  
	  
		char msg[255];  
	  
		int partmask = 0;  
		int i;  
		for (i = 1; i <= mDiskNumParts; i++) {  
		    partmask |= (1 << i);  
		}  
		mPendingPartMap = partmask;  
	  
		if (mDiskNumParts == 0) {  
	#ifdef PARTITION_DEBUG  
		    SLOGD("Dv::diskIns - No partitions - good to go son!");  
	#endif  
		    setState(Volume::State_Idle);  
		} else {  
	#ifdef PARTITION_DEBUG  
		    SLOGD("Dv::diskIns - waiting for %d partitions (mask 0x%x)",  
		         mDiskNumParts, mPendingPartMap);  
	#endif  
		    setState(Volume::State_Pending);  
		}  
	  
		snprintf(msg, sizeof(msg), "Volume %s %s disk inserted (%d:%d)",  
		         getLabel(), getMountpoint(), mDiskMajor, mDiskMinor);  
		mVm->getBroadcaster()->sendBroadcast(ResponseCode::VolumeDiskInserted,  
		                                         msg, false);  
	}  

mDiskNumParts 不为0，将Volume的状态设置为State_Pending并向FrameWork层广播VolumeDiskInserted的消息，
在setState函数中也会广播VolumeStateChange的消息给上层，接着就是handlePartitionAdded 
这里是处理add /block/sda/sda*这样的消息的

	void DirectVolume::handlePartitionAdded(const char *devpath, NetlinkEvent *evt) {  
		int major = atoi(evt->findParam("MAJOR"));  
		int minor = atoi(evt->findParam("MINOR"));  
	  
		int part_num;  
	  
		const char *tmp = evt->findParam("PARTN");  
	  
		if (tmp) {  
		    part_num = atoi(tmp);  
		} else {  
		    SLOGW("Kernel block uevent missing 'PARTN'");  
		    part_num = 1;  
		}  
	  
		if (part_num > mDiskNumParts) {  
		    mDiskNumParts = part_num;  
		}  
	  
		if (major != mDiskMajor) {  
		    SLOGE("Partition '%s' has a different major than its disk!", devpath);  
		    return;  
		}  
	#ifdef PARTITION_DEBUG  
		SLOGD("Dv:partAdd: part_num = %d, minor = %d\n", part_num, minor);  
	#endif  
		mPartMinors[part_num -1] = minor;  
	  
		mPendingPartMap &= ~(1 << part_num);  
		if (!mPendingPartMap) {  
	#ifdef PARTITION_DEBUG  
		    SLOGD("Dv:partAdd: Got all partitions - ready to rock!");  
	#endif  
		    if (getState() != Volume::State_Formatting) {  
		        setState(Volume::State_Idle);  
		    }  
		} else {  
	#ifdef PARTITION_DEBUG  
		    SLOGD("Dv:partAdd: pending mask now = 0x%x", mPendingPartMap);  
	#endif  
		}  
	}  


当mPendingPartMap减为0时，这时Volume的状态不为State_Formatting，将广播一条VolumeStateChange的消息。
到这里，内核的消息基本就处理完了，当然这里讲的只是add的消息，还有remove，change消息等。。。这里就不做介绍了









## 附一 参考资料
1	android usb挂载分析----vold启动		
	http://blog.csdn.net/new_abc/article/details/7396733		
2	android usb挂载分析---MountService启动		
	http://blog.csdn.net/new_abc/article/details/7400740		
3	android usb挂载分析---vold处理内核消息		
	http://blog.csdn.net/new_abc/article/details/7409018		
4	android usb挂载分析---FrameWork层处理收到的vold消息		
	http://blog.csdn.net/new_abc/article/details/7413317		
5	android usb挂载分析---vold处理FrameWork层发出的消息		
	http://blog.csdn.net/new_abc/article/details/7413452		
6	android usb挂载分析---FrameWork层处理vold消息		
	http://blog.csdn.net/new_abc/article/details/7417539		
7	android usb挂载分析--类图		
	http://blog.csdn.net/new_abc/article/details/7420755		




















