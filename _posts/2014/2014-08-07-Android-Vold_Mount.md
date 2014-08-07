---
layout: post
title: Android Vold & Mount
categories:
- programmer
tags:
- android
---


http://blog.csdn.net/new_abc/article/details/8647124


## Vold 启动

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
	  
			//mCtrlPipe[0]有数据，则结循环，注意是在stopListener的时候 往mCtrlPipe[1]写数据 
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


Ok，到这里，vold基本就启动起来了，基本的通信环境也已经搭建好了，就等着u盘插入后kernel的消息的











1	android usb挂载分析----vold启动
	http://blog.csdn.net/new_abc/article/details/7396733



















