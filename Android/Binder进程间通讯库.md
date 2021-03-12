### Binder进程间通讯库

Android系统在应用程序框架层中将各种Binder驱动程序操作封装成一个Binder库，这样进程就可以方便地调用Binder库提供的接口来实现进程间通信。

在Binder库中，**Service组件和Client组件分别使用模板类BnInterface和BpInterface来描述**，其中前者为Binder本地对象，后者为Binder代理对象。**Binder库中的Binder本地对象和Binder代理对象分别对应于Binder驱动程序中的Binder实体对象和Binder引用对象**。

模板类BnInterface和BpInterface的定义如下：

```c++
frameworks/base/include/binder/IInterface.h
template<typename INTERFACE> //模板参数INTERFACE是一个由进程自定义的Service组件接口
class BnInterface : public INTERFACE, public BBinder{ //描述Service组件
  public:
  			virtual sp<IInterface>  queryLocalInterface(const String16& _descriptor);
  			virtual const String16& getInterfaceDescriptor() const;
  protected:
  		virtual IBinder*          onAsBinder();
};

template<typename INTERFACE>
class BpInterface : public INTERFACE, public BpRefBase{  //描述Client组件
  public:
  				BpInterface(const sp<IBinder>& remote);
  protected:
  				virtual IBinder*  onAsBinder();
}


```

在开发Service组件和Client组件时，**除了要定义Service组件接口之外，还必须要实现一个Binder本地对象类和一个Binder代理对象类**,它们分别继承于模板类BnInterface和BpInterface

```c++
frameworks/base/include/binder/Binder.h
//模板类BnInterface继承了BBinder,BBinder为Binder本地对象提供了抽象的进程间通信接口
class BBinder : public IBinder{  
  public:
  	...
    virtual status_t transact(uint32_t code,
                              const Parcel& data,
                              Parcel* reply,
                              uint32_t flags = 0);
  protected:
  		...
    	virtual status_t onTransact(uint32_t code,
                                  const Parcel& data,
                                  Parcel* reply,
                                  uint32_t flags = 0)
}
```

当一个Binder代理对象通过Binder驱动程序向一个Binder本地对象发出一个进程间通信请求时**，Binder驱动程序就会调用对应的Binder本地对象的成员函数`transact`来处理该请求。**

成员函数`onTransact`是BBinder的子类，即Binder本地对象类来实现的，它负责分发与业务相关的进程间通信请求(类比layout和onLayout的方式)

BBinder类又继承了IBinder类，**而IBinder又继承了RefBase类。继承了RefBase类的子类的对象均可以通过强指针和弱指针来维护它们的生命周期**。也就是说Binder本地对象是通过引用计数来维护生命周期的

```c++
frameworks/base/include/binder/Binder.h
//模板类BpInterface继承了BpRefBase
//BpRefBase为Binder代理对象提供了抽象的进程间通信接口
//而BpRefBase又继承了RefBase类，因此它的子类对象，即Binder代理对象也可以通过强指针和弱指针来维护生命周期
class BpRefBase : public virtual RefBase{  
  protected:
               BpRefBase(const sp<IBinder>& o);
  inline IBinder* remote()   {return mRemote;}
  inline IBinder* remote()  const  {return mRemote;}
  
  private:
  	...
  IBinder* const mRemote; //mRemote指向一个BpBinder对象，可以通过成员函数remote来获取
};
```

```c++
frameworks/base/include/binder/BpBinder.h
//BpBinder类实现了BpRefBase类的进程间通信接口
class BpBinder : public IBinder{
  public: 
  			BpBinder(int32_t handle);
  inline int32_t       handle() const {return mHandle;}
  virtual status_t     transact(uint32_t code,
                                const Parcel& data,
                                Parcel* reply,
                                uint32_t flags = 0);
  private: 
     const int32_t      mHandle; //表示Client组件的句柄值，可以通过成员函数handle获取
};
```

Client组件就是通过这个句柄值来和Binder驱动程序中的Binder引用对象建立对应关系的。

**`BpBinder`类的成员函数`transact`用来向运行在Server进程中的Service组件发送进程间通信请求，transact会把BpBinder类的成员变量`mHandle`,以及进程间通信数据发送给Binder驱动程序**，这样Binder驱动程序就能够根据这个句柄值来找到对应的Binder引用对象，继而找到对应的Binder实体对象。

无论是BBinder类还是BpBinder类，**它们都是通过IPCThreadState类来和Binder驱动程序交互的**。

```c++
frameworks/base/include/binder/IPCThreadState.h
class IPCThreadState{
  public: 
  		static IPCThreadState*     self();
  		......
    	status_t                   transact(int32_t handle,
                                          unint32_t code,
                                          const Parcel& data,
                                          Parcel* reply,
                                          uint32_t flags);
  private:
      status_t                   talkWithDriver(bool doReceive = true);
 		 	......
    	const   sp<ProcessState>     mProcess;
}
```

**每一个使用了Binder通信机制的进程都有一个Binder线程池**，用来处理进程间通信请求。**对于每一个Binder线程来说，内部都有一个IPCThreadState对象**，可以通过IPCThreadState类的静态成员函数self来获取，并且调用成员**函数transact来和Binder驱动程序交互，在IPCThreadState类的成员函数transact内部，与Binder驱动程序交互操作又是通过调用成员函数talkWithDriver来实现的。**

> IPCThreadState类有一个成员变量mProcess,它指向一个ProcessState对象，对于每一个使用了Binder进程间通信机制的进程来说，它的内部都有一个ProcessState对象，负责初始化Binder设备，即打开设备文件/dev/binder,以及将设备文件/dev/binder映射到进程的地址空间。由于这个ProcessState对象在进程范围内是唯一的。因此，Binder线程池中的每一个线程都可以通过它来和Binder驱动程序建立连接。

```c++
frameworks/base/include/binder/ProcessState.h
class ProcessState : public virtual RefBase{
  public:
  		static sp<ProcessState>   self();
  private:
      int             mDriverFD;
  		void*           mVMStart;
}
```

进程中的ProcessState对象可以通过ProcessState类的静态成员函数self来获取。第一次调用self函数时，**Binder库就会为进程创建一个ProcessState对象，并且调用函数open来打开设备文件/dev/binder,接着又调用函数mmap将它映射到进程的地址空间，即请求Binder驱动程序为进程分配内核缓冲区**。设备文件/dev/binder映射到进程的地址空间后，得到的内核缓冲区的用户地址就保存在其成员变量mVMStart中。

### Binder进程间通信实例

包含一个Server进程和一个Client进程，其中Server进程实现了一个Service组件

```shell
~Andorid/external/binder
----common
	----IFregService.h
	----IFregService.cpp
----server
	----FregService.cpp
	----Android.mk
----client
	----FregClient.cpp
	----Android.mk
```

```c++
common/IFregService.h
#ifndef IFREGSERVICE_H_
#define IFREGSERVICE_H_

#include <utils/RefBase.h>
#include <binder/IInterface.h>
#include <binder/Parcel.h>
  
#define FREG_SERVICE "chong.zhang.FregService"  //注册到ServiceManager的名称
  
using namespace android;

class IFregService: public IInterface{ //定义硬件访问服务接口
  public:
  		virtual int32_t getVal() = 0;
  		virtual void setVal()(int32_t val) = 0;
};

//实现了模板类成员函数BnInterfaceonTranscat
class BnFregService: public BnInterface<IFregService>{ 
  public:
  		virtual status_t onTranscat(uint32_t code,const Parcel& data,Parcel* reply,uint32_t flags = 0);
};

#endif
```

```c++
common/IFregService.cpp
  #define LOG_TAG "IFregService"
  
  #include <utils/Log.h>
  #include "IFregService.h"
  
  using namespace android;

enum{
  GET_VAL = IBinder::FIRST_CALL_TRANSCATION,
 	SET_VAL
};

class BpFregService: public BpInterface<IFregService>{
  public:
  		BpFregService(const sp<IBinder>& impl):BpInterface<IFregService>(impl){}
  public:
  		int32_t getVal(){
        Parcel data;
        data.writeInterfaceToken(IFregService::getInterfaceDescriptor())
          
        Parcel reply;
        remote()->transact(GET_VAL,data,$reply);
        
        int32_t val = reply.readInt32();
        
        return val;
      }
  
  		void setVal(int32_t val){
        Parcel data;
        data.writeInterfaceToken(IFregService::getInterfaceDescriptor());
        data.writeInt32(val);
        
        Parcel reply;
        remote() -> transact(SET_VAL,data,&reply);
      }
};

status_t BnFregService::onTransact(uint32_t code,const Parcel& data,Parcel* reply,uint32_t flags){
  switch(code){
    case GET_VAL:{
      int32_t val = getVal();
      reply->writeInt32(val);
      return NO_ERROR;
    }
      
    case SET_VAL:{
      int32_t val = data.readInt32();;
      setVal(val);
      
      return NO_ERROR;
    }
  }
  
}


```

```c++
/server/FregService.cpp
#define LOG_TAG "FregServer"

#include <stdlib.h>
  
#include <binder/IServiceManager.h>
#include <binder/IPCThreadState.h>
  
#include "../common/IFregService.h"
  
class FregService : public BnFregService{
  public: 
  		FregService(){
        fd = open(FREG_DEVICE_NAME, O_RDWR);
        if(fd == -1){
          LOGE("Failed to open device %s.\n",FREG_DEVICE_NAME);
        }
      }
  
  		virtual ~FregService(){
        		if(fd != -1){
                 close(fd);
            }
      }
  
  public: 
  		static void instantiate(){
        defaultServiceManager()->addService(String16(FRGG_SERVICE),new FregService());
      }
  
  		int32_t getVal(){
        		int32_t val = 0;
        		if(fd != -1){
              		read(fd, &val, sizeof(val));
            }
        		return val;
      }
  
  		void setVal(int32_t val){
        if(fd != -1){
          		write(fd,&val,sizeof(val));
        }
      }
  
  private:
  		int fd;
  
};

int main(int argc, char** argv){
  	FregService::instantiate(); //将Service组件注册到ServiceManager
  
  	ProcessState::self()->startThreadPool(); //启动一个Binder线程池
  	IPCThreadState::self()->joinThreadPool(); //将主线程添加到进程的Binder线程池中，用来处理来自Client进程的通信请求
  
  	return 0;
}



```





