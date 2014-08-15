---
layout: post
title: Android Vold & Mount
categories:
- programmer
tags:
- android
---


android 4.4.2

![Alt text](http://zhongguomin.github.io/blog/media/images/2014/Android-Vold_Mount-01.jpg "Android-Vold_Mount-01.jpg")


## Vold 启动

	VolumeManager
	NetlinkManager
	CommandListener
	FrameworkListener
	NetlinkHandler
	NetlinkListener


vold启动在init.rc中：

	service vold /system/bin/vold
		class core
		socket vold stream 0660 root mount
		ioprio be 2


注意这里创建了一个socket，用于vold和Framework层通信


vold入口main函数：

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

这里首先创建了CommandListener,CommandListener主要负责与Framework层的通信，
处理从Framework层收到的各种命令

	CommandListener::CommandListener() :
		             FrameworkListener("vold", true) {
		registerCmd(new DumpCmd());
		registerCmd(new VolumeCmd());
		registerCmd(new AsecCmd());
		registerCmd(new ObbCmd());
		registerCmd(new StorageCmd());
		registerCmd(new XwarpCmd());
		registerCmd(new CryptfsCmd());
		registerCmd(new FstrimCmd());
	}

	FrameworkListener::FrameworkListener(const char *socketName, bool withSeq) :
		                        SocketListener(socketName, true, withSeq) {
		init(socketName, withSeq);
	}

这里注册了各种命令，注意FrameworkListener("vold")，FrameworkListener又继承了SocketListener，
最终"vold"传到了SocketListener里面。

	vm->setBroadcaster((SocketListener *) cl);
	nm->setBroadcaster((SocketListener *) cl);

	void setBroadcaster(SocketListener *sl) { mBroadcaster = sl; }

设置了Broadcaster，后面给Framework层发送消息就跟它有关了

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

		//注册UEVENT事件，用于接收内核消息
		if ((mSock = socket(PF_NETLINK,
		                    SOCK_DGRAM,NETLINK_KOBJECT_UEVENT)) < 0) {
		    SLOGE("Unable to create uevent socket: %s", strerror(errno));
		    return -1;
		}
		
		if (setsockopt(mSock, SOL_SOCKET, SO_RCVBUFFORCE, &sz, sizeof(sz)) < 0) {
		    SLOGE("Unable to set uevent socket SO_RCVBUFFORCE option: %s", strerror(errno));
		    goto out;
		}

		if (setsockopt(mSock, SOL_SOCKET, SO_PASSCRED, &on, sizeof(on)) < 0) {
		    SLOGE("Unable to set uevent socket SO_PASSCRED option: %s", strerror(errno));
		    goto out;
		}
		    
		if (bind(mSock, (struct sockaddr *) &nladdr, sizeof(nladdr)) < 0) {
		    SLOGE("Unable to bind uevent socket: %s", strerror(errno));
		    goto out;
		}
		
		//NetlinkHandler用于对接收到的内核消息进行处理 
		mHandler = new NetlinkHandler(mSock);
		//开始监听内核消息 
		if (mHandler->start()) {
		    SLOGE("Unable to start NetlinkHandler: %s", strerror(errno));
		    goto out;
		}
		
		return 0;
		
	out:
		close(mSock);
		return -1;
	}

我们跟进mHandler->start()最终调用 SocketListener::startListener()

	NetlinkHandler -> NetlinkListener ->  SocketListener

	int SocketListener::startListener() {

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
		    SLOGV("got mSock = %d for %s", mSock, mSocketName);
		}

		if (mListen && listen(mSock, 4) < 0) {
		    SLOGE("Unable to listen on socket (%s)", strerror(errno));
		    return -1;
		} else if (!mListen)
		    mClients->push_back(new SocketClient(mSock, false, mUseCmdNum));

		if (pipe(mCtrlPipe)) {
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

	void *SocketListener::threadStart(void *obj) {
		SocketListener *me = reinterpret_cast<SocketListener *>(obj);

		me->runListener();
		pthread_exit(NULL);
		return NULL;
	}


	void SocketListener::runListener() {

		SocketClientCollection *pendingList = new SocketClientCollection();

		while(1) {
		    SocketClientCollection::iterator it;
		    fd_set read_fds;
		    int rc = 0;
		    int max = -1;

		    FD_ZERO(&read_fds);

		    if (mListen) {
		        max = mSock;
		        FD_SET(mSock, &read_fds);
		    }

			// Note: mCtrlPipe
		    FD_SET(mCtrlPipe[0], &read_fds);
		    if (mCtrlPipe[0] > max)
		        max = mCtrlPipe[0];

		    pthread_mutex_lock(&mClientsLock);
		    for (it = mClients->begin(); it != mClients->end(); ++it) {
		        int fd = (*it)->getSocket();
		        FD_SET(fd, &read_fds);
		        if (fd > max)
		            max = fd;
		    }
		    pthread_mutex_unlock(&mClientsLock);
		    SLOGV("mListen=%d, max=%d, mSocketName=%s", mListen, max, mSocketName);

			// select
		    if ((rc = select(max + 1, &read_fds, NULL, NULL, NULL)) < 0) {
		        if (errno == EINTR)
		            continue;
		        SLOGE("select failed (%s) mListen=%d, max=%d", strerror(errno), mListen, max);
		        sleep(1);
		        continue;
		    } else if (!rc)
		        continue;

		    if (FD_ISSET(mCtrlPipe[0], &read_fds))
		        break;
		    if (mListen && FD_ISSET(mSock, &read_fds)) {
		        struct sockaddr addr;
		        socklen_t alen;
		        int c;

		        do {
		            alen = sizeof(addr);
					// accept
		            c = accept(mSock, &addr, &alen);
		            SLOGV("%s got %d from accept", mSocketName, c);
		        } while (c < 0 && errno == EINTR);
		        if (c < 0) {
		            SLOGE("accept failed (%s)", strerror(errno));
		            sleep(1);
		            continue;
		        }
		        pthread_mutex_lock(&mClientsLock);
		        mClients->push_back(new SocketClient(c, true, mUseCmdNum));
		        pthread_mutex_unlock(&mClientsLock);
		    }

		    /* Add all active clients to the pending list first */
		    pendingList->clear();
		    pthread_mutex_lock(&mClientsLock);
		    for (it = mClients->begin(); it != mClients->end(); ++it) {
		        int fd = (*it)->getSocket();
		        if (FD_ISSET(fd, &read_fds)) {
		            pendingList->push_back(*it);
		        }
		    }
		    pthread_mutex_unlock(&mClientsLock);

		    /* Process the pending list, since it is owned by the thread,
		     * there is no need to lock it */
		    while (!pendingList->empty()) {
		        /* Pop the first item from the list */
		        it = pendingList->begin();
		        SocketClient* c = *it;
		        pendingList->erase(it);
		        /* Process it, if false is returned and our sockets are
		         * connection-based, remove and destroy it */

				// onDataAvailable
		        if (!onDataAvailable(c) && mListen) {
		            /* Remove the client from our array */
		            SLOGV("going to zap %d for %s", c->getSocket(), mSocketName);
		            pthread_mutex_lock(&mClientsLock);
		            for (it = mClients->begin(); it != mClients->end(); ++it) {
		                if (*it == c) {
		                    mClients->erase(it);
		                    break;
		                }
		            }
		            pthread_mutex_unlock(&mClientsLock);
		            /* Remove our reference to the client */
		            c->decRef();
		        }
		    }
		}
		delete pendingList;
	}

这样，就开始了监听来自内核的事件

	coldboot("/sys/block");
	//coldboot("/sys/class/switch");

    /*
     * Now that we're up, we can respond to commands
     */
    if (cl->startListener()) {
        SLOGE("Unable to start CommandListener (%s)", strerror(errno));
        exit(1);
    }

这里主要看cl->startListener,也跟前面的一样，调用SocketListener::startListener()，注意这时

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
OK，到这里，vold基本就启动起来了，基本的通信环境也已经搭建好了，就等着u盘插入后kernel的消息的



## MountService启动
vold模块已经启动了，通信的机制也已经建立起来了，接下来我们分析一下MountService的启动，
也就是我们Framework层的启动

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

这里new了一个MountService，并把service添加到了ServiceManager,我们看下MountService的构造函数：

	/** 
	 * Constructs a new MountService instance 
	 * 
	 * @param context  Binder context for this service 
	 */
	public MountService(Context context) {
		mContext = context;  
	  
		// XXX: This will go away soon in favor of IMountServiceObserver  
		mPms = (PackageManagerService) ServiceManager.getService("package");
	  
		mContext.registerReceiver(mBroadcastReceiver,  
		        new IntentFilter(Intent.ACTION_BOOT_COMPLETED), null, null);
	  
		//处理消息
		mHandlerThread = new HandlerThread("MountService");
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

后面new了一个NativeDaemonConnector，注意这里传递了一个"vold"字符串，
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
		                if (LOCAL_LOGD) Slog.d(TAG, String.format("RCV <- (%s)", event));
	  
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

这样，就建立了一条从Framework层到vold层的通信链路，后面Framework层就等待Vold发送消息过来了。
Framework层的通信也ok了，就可以等待U盘挂载了。。




## vold处理内核消息

这里要讲的是内核发信息给vold，我们在 vold启动讲到过注册了一个到内核的UEVENT事件，当有u盘插入的时候，
我们就能从这个套接字上收到内核所发出的消息了，这样就开始了vold的消息处理

	SocketListener
	NetlinkListener
	NetlinkEvent
	NetlinkHandler
	VolumeManager
	DiectVolume


在SocketListener::runListener()函数中，我们一直在select，等待某个连接的到来或者已经的套接字上数据的到来


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

mDiskNumParts 不为0，将Volume的状态设置为State_Pending并向Framework层广播VolumeDiskInserted的消息，
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



## Framework层处理收到的vold消息
vold模块收到内核消息后，通过前面建立的socket通信各上去发送相应的消息，我们可以看到主要发了两类消息：			
1、DirectVolume::handleDiskAdded以及handlePartitionAdded都调用setState发送了一条			
	VolumeStateChange消息。		
2、handleDiskAdded中还发送了 VolumeDiskInserted消息。		
我们先看下Framework层的消息处理流程：		


	NativeDaemonConnection
	MountService


这里的消息处理还算比较简单，主要只要处理VolumeDiskInserted消息，另外两条消息收到后都被忽略了，
我们看下VolumeDiskInserted的处理，首先阻塞在listenToSocket等待vold消息的到来：


	while (true) {
		int count = inputStream.read(buffer, start, BUFFER_SIZE - start);  
		if (count < 0) break;  

		try {  
			if (!mCallbacks.onEvent(code, event, tokens)) {  
				Slog.w(TAG, String.format(  
					"Unhandled event (%s)", event));  
			}
		}

收到消息后，调用onEvent函数
onEvent的函数实现在MountService中


	public boolean onEvent(int code, String raw, String[] cooked) {
		Intent in = null;  

		if (code == VoldResponseCode.VolumeDiskInserted) {  
			new Thread() {
				public void run() {  
					try {  
						int rc;  
						if((rc = doMountVolume(path))!=StorageResultCode.OperationSucceeded) {
							Slog.w(TAG, String.format("Insertion mount failed (%d)", rc));  
						}  
					} catch (Exception ex) {  
						Slog.w(TAG, "Failed to mount media on insertion", ex);  
					}  
				}  
			}.start();

这里的消息为VolumeDiskInserted，new 一个Thread并start，
在run函数中调用doMountVolume函数向vold层发送挂载命令：

	private int doMountVolume(String path) {
		int rc = StorageResultCode.OperationSucceeded;  
  
		if (DEBUG_EVENTS) Slog.i(TAG, "doMountVolume: Mouting " + path);  
		try {  
			mConnector.doCommand(String.format("volume mount %s", path));  
		}
	}

这里调用doCommand并以volume mount path为参数，我们看下doCommand：

	public synchronized ArrayList<String> doCommand(String cmd)
		throws NativeDaemonConnectorException  {  
		sendCommand(cmd);  

	}

继续看sendCommand：

	private void sendCommand(String command, String argument)
		throws NativeDaemonConnectorException  {  

		try {  
			mOutputStream.write(builder.toString().getBytes());  
		} catch (IOException ex) {  
			Slog.e(TAG, "IOException in sendCommand", ex);  
		}

	}

调用write函数把消息发送到vold层，这样Framework层就把挂载命令下发到了vold层




## vold处理Framework层发出的消息

Framework层收到消息后，又向vold发送了volume mount的消息，所以vold层又继续着处理这个消息,先看下大概处理流程：


	SocketListener
	FrameworkListener
	CommandListener
	Volume
	Fat


同Framework层阻塞在等待vold的消息一样，vold层也在等待着收到 Framework层的消息，
不过是调用select函数百阻塞，因为这个还有内核可能会有其它的连接请求的到来等，所以不能阻塞。


	void SocketListener::runListener() {

	    if ((rc = select(max + 1, &read_fds, NULL, NULL, NULL)) < 0) {
			SLOGE("select failed (%s)", strerror(errno));  
            sleep(1);  
            continue;  
        }   

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


收到消息后，调用onDataAvailable，这里这个函数的实现是在FrameworkListener类中，
在onDataAvailable中接收数据，并调用dispatchCommand对分发命令：


	void FrameworkListener::dispatchCommand(SocketClient *cli, char *data) {
		.  
		.  
		for (i = mCommands->begin(); i != mCommands->end(); ++i) {  
		    FrameworkCommand *c = *i;
	  
		    if (!strcmp(argv[0], c->getCommand())) {
		        if (c->runCommand(cli, argc, argv)) {
		            SLOGW("Handler '%s' error (%s)", c->getCommand(), strerror(errno));
		        }
		        goto out;
		    }
		}  
	}

mCommands中的命令是在什么时候加进去的？回顾下CommandListener的初始化，我们注册了很多的命令，
对的，就是在注册这些命令的时候加进去的，这里传下来的命令是volume mount ,所以调用 VolumeCmd::runCommand


	int CommandListener::VolumeCmd::runCommand(SocketClient *cli,
		int argc, char **argv) {  
		.  
		.  
		else if (!strcmp(argv[1], "mount")) {  
			if (argc != 3) {  
				cli->sendMsg(ResponseCode::CommandSyntaxError, 
					"Usage: volume mount <path>", false);  
		        return 0;
			}  
			rc = vm->mountVolume(argv[2]);  
	} 

针对mount命令，调用mountVolume，mountVolume中继续调用mountVol：

	int Volume::mountVol() {
		dev_t deviceNodes[4];  
		int n, i, rc = 0;  
		char errmsg[255];  
	  
		if (getState() == Volume::State_NoMedia) {  
		    snprintf(errmsg, sizeof(errmsg),
		             "Volume %s %s mount failed - no media",
		             getLabel(), getMountpoint());
		    mVm->getBroadcaster()->sendBroadcast(
					ResponseCode::VolumeMountFailedNoMedia,  
		            errmsg, false);
		    errno = ENODEV;
		    return -1;
		} else if (getState() != Volume::State_Idle) {  
		    errno = EBUSY;
		    return -1;
		}  
	  
		if (isMountpointMounted(getMountpoint())) {  
		    SLOGW("Volume is idle but appears to be mounted - fixing");
		    setState(Volume::State_Mounted);
		    // mCurrentlyMountedKdev = XXX
		    return 0;
		}  
	  
		n = getDeviceNodes((dev_t *) &deviceNodes, 4);  
		if (!n) {  
		    SLOGE("Failed to get device nodes (%s)\n", strerror(errno));
		    return -1;
		}  
	  
		for (i = 0; i < n; i++) {  
		    char devicePath[255];
		    int result = 0;
		    const char *disktype = "fat";
	  
		    sprintf(devicePath, "/dev/block/vold/%d:%d", MAJOR(deviceNodes[i]),
		            MINOR(deviceNodes[i]));
	  
		    SLOGI("%s being considered for volume %s\n", devicePath, getLabel());
	  
		    errno = 0;
		    setState(Volume::State_Checking);
	  
		    result = Fat::check(devicePath);
		    if(result)
		    {
		        result = Ntfs::check(devicePath);
		        if(!result)
		        {
		             disktype = "ntfs";
		        }
		    }
		           
		    if (result) {
		        if (errno == ENODATA) {
		            SLOGW("%s does not contain a FAT(Ntfs) filesystem\n", devicePath);
		            continue;
		        }
		        errno = EIO;
		        /* Badness - abort the mount */
		        SLOGE("%s failed FS checks (%s)", devicePath, strerror(errno));
		        setState(Volume::State_Idle);
		        return -1;
		    }
	  
		    /* 
		     * Mount the device on our internal staging mountpoint so we can 
		     * muck with it before exposing it to non priviledged users. 
		     */
		    errno = 0;
		    if(0 == strcmp(disktype, "fat"))
		    {
		        if (Fat::doMount(devicePath, "/mnt/secure/staging", false, 
					false,false, 1000, 1015, 0702, true)) {  
		            SLOGE("%s failed to mount via VFAT (%s)\n", devicePath, strerror(errno));
		            continue;
		        }
		    }
		    else if(0 == strcmp(disktype, "ntfs"))
		    {
		        if (Ntfs::doMount(devicePath, "/mnt/secure/staging", false, 
					false,false, 1000, 1015, 0702, true)) {  
		            SLOGE("%s failed to mount via NTFS (%s)\n", devicePath, strerror(errno));
		            continue;
		        }
		    }
	  
		    SLOGI("Device %s, target %s mounted @ /mnt/secure/staging", devicePath, 
				getMountpoint());  
	  
		    protectFromAutorunStupidity();
	  
		    if (createBindMounts()) {
		        SLOGE("Failed to create bindmounts (%s)", strerror(errno));
		        umount("/mnt/secure/staging");
		        setState(Volume::State_Idle);
		        return -1;
		    }
	  
		    /* 
		     * Now that the bindmount trickery is done, atomically move the 
		     * whole subtree to expose it to non priviledged users. 
		     */
		    if (doMoveMount("/mnt/secure/staging", getMountpoint(), false)) {
		        SLOGE("Failed to move mount (%s)", strerror(errno));
		        umount("/mnt/secure/staging");
		        setState(Volume::State_Idle);
		        return -1;
		    }
		    setState(Volume::State_Mounted);
		    mCurrentlyMountedKdev = deviceNodes[i];
		    return 0;
		}  
	  
		SLOGE("Volume %s found no suitable devices for mounting :(\n", getLabel());  
		setState(Volume::State_Idle);  
	  
		return -1;  
	}


mountVol中首先检票Volume的状态，这里面必须为State_Idle状态才会进行后面的操作，

这里有一点需要注意下，我们知道，在DirectVolume::handleDiskAdded的时候 向Framework层发送VolumeDiskInserted消息，这个时候 Framework层才下发volume mount消息，但是这个时候Voleme的State为State_Pending,要等到内核将这块设备的所有分区的add消息发出并调用完handlePartitionAdded才将Volume的状态设为State_Idle，这里会不会发生这种情况：Framework消息已经发下来了要进行mount了，但add分区的消息还没处理完，这个时候Volume的状态仍为State_Pending，所以在这里mountVol检查状态的时候不正确，直接返回失败，
因为在我们的项目中发现有的时候存储设备会挂载不上，所以这里加了一个延时处理，状态不对时，睡眠一会再处理。状态检查之后调用getDeviceNodes获取有多少分区，然后对所有分区一一进行挂载，

注意挂载的时候是先挂载到/mnt/secure/staging，然后现调用doMoveMount移动到挂载点。



## Framework层处理vold消息


从前面的知识我们看到，在vold层收到 Framework层的消息后，会进行相应的处理，同时在处理的过程中会上报相应的状态给Framework层，在这个过程中主要上报了两种消息：
1、开始挂载前上报State_Checking消息。
2、挂载成功后上报State_Mounted消息。
针对这两个消息，我们看下Framework层相应的处理，这两个消息处理的流程基本差不多，只是对于State_Mounted在处理的时候多了一个updateExternalMediaStatus通知PackageManagerService进行相应的更新

	MountServicec

首先还是阻塞在NativeDaemonConnector中的listenToSocket等待vold层消息的到来，收到消息后，调用onEvent函数进行处理，这里收到的这两个消息类型都是VolumeStateChange，所以调用notifyVolumeStateChange函数


对Checking和Mount两类消息都是调用updatePublicVolumeState进行处理

针对Mount消息会调用updateExternalMediaStatus去更新PackageManagerService中的一些信息


调用onStorageStateChanged通知SDCARD状态改变，这个listener是哪里来的呢，我们跟踪源码可以看到是MountServiceBinderListener的一个成员变量，其类型是IMountServiceListener，IMountServiceListener是一个接口，我们跟踪其实现，最后可以看到是在StorageManager的MountServiceBinderListener的这个类继承了IMountServiceListener这个接口，而且我们看StorageManager的构造函数：

	public StorageManager(Looper tgtLooper) throws RemoteException {
		mMountService = IMountService.Stub.asInterface(ServiceManager.getService("mount"));  
		if (mMountService == null) {  
		    Log.e(TAG, "Unable to connect to mount service! - is it running yet?");
		    return;
		}  
		mTgtLooper = tgtLooper;  
		mBinderListener = new MountServiceBinderListener();  
		mMountService.registerListener(mBinderListener);  
	}

对的，就是在它的构造函数中实例化了一个listener并注册到MountService中，回到上面，调用了listener的onStorageStateChanged，这里通过binder最终调用了StorageManager类中MountServiceBinderListener的onStorageStateChanged：

	public void onStorageStateChanged(String path, String oldState, String newState) {
		final int size = mListeners.size();  
		for (int i = 0; i < size; i++) {  
		    mListeners.get(i).sendStorageStateChanged(path, oldState, newState);
		}  
	}

这里的mListeners是一个List,其中的成员是ListenerDelegate类，所以这里调用了ListenerDelegate的sendStorageStateChanged方法

	void sendStorageStateChanged(String path, String oldState, String newState) {
		StorageStateChangedStorageEvent e = 
			new StorageStateChangedStorageEvent(path, oldState, newState);  
		mHandler.sendMessage(e.getMessage());  
	}

这里只是简单的发一条StorageStateChangedStorageEvent 消息，我们看看下这个Handle 的处理函数，在ListenerDelegate类中重写了handleMessage方法：

	public void handleMessage(Message msg) {
		StorageEvent e = (StorageEvent) msg.obj;  
	  
		if (msg.what == StorageEvent.EVENT_UMS_CONNECTION_CHANGED) {  
			UmsConnectionChangedStorageEvent ev = (UmsConnectionChangedStorageEvent) e;  
			mStorageEventListener.onUsbMassStorageConnectionChanged(ev.available);  
		} else if (msg.what == StorageEvent.EVENT_STORAGE_STATE_CHANGED) {  
			StorageStateChangedStorageEvent ev = (StorageStateChangedStorageEvent) e;  
			mStorageEventListener.onStorageStateChanged(ev.path, ev.oldState, ev.newState);  
		} else {  
			Log.e(TAG, "Unsupported event " + msg.what);  
		}  
	}
	
这里调用mStorageEventListener的onStorageStateChanged，这里mStorageEventListener是StorageEventListener类型的。
一般应用需要在sd卡状态改变的时候做一些处理也就只要继承StorageEventListener并重写mStorageEventListener这个方法，然后把这个Listener  调用StorageManager 的registerListener注册进来，这样在sdcard状态变化的时候就能收到消息了，进行处理了。
好了，对于vold的发的这两个消息的处理就差不多了 State_Checking和State_Mounted处理的流程基本都是一样的，只是对于State_Mounted消息还多了一个和PackageManageService打交道的过程，我们最后来看下这个交互过程都做了什么：

	public void updateExternalMediaStatus(final boolean mediaStatus, final 
		boolean reportStatus) {  
		if (Binder.getCallingUid() != Process.SYSTEM_UID) {  
		    throw new SecurityException("Media status can only be updated by the system");
		}  
		synchronized (mPackages) {  
		    Log.i(TAG, "Updating external media status from " +
		            (mMediaMounted ? "mounted" : "unmounted") + " to " +
		            (mediaStatus ? "mounted" : "unmounted"));
		    if (DEBUG_SD_INSTALL) Log.i(TAG, "updateExternalMediaStatus:: mediaStatus=" +
		            mediaStatus+", mMediaMounted=" + mMediaMounted);
		    if (mediaStatus == mMediaMounted) {
		        Message msg = mHandler.obtainMessage(UPDATED_MEDIA_STATUS,
		                reportStatus ? 1 : 0, -1);
		        mHandler.sendMessage(msg);
		        return;
		    }
		    mMediaMounted = mediaStatus;
		}  
		// Queue up an async operation since the package installation may take a little while.  
		mHandler.post(new Runnable() {  
		    public void run() {
		        mHandler.removeCallbacks(this);
		        updateExternalMediaStatusInner(mediaStatus, reportStatus);
		    }
		});  
	}

这里主要是调用updateExternalMediaStatusInner，这里面又主要调用loadMediaPackages函数，我们看下这个函数：
这里面主要是调用scanPackageLI对sd卡里面的东西进行处理。



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
8	Android Vold架构		
	http://blog.csdn.net/qianjin0703/article/details/6362389		



















