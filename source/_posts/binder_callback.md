---
title: 【一起来学binder】java/cpp端回调的简单示例
date: 2021-01-26 23:22:27
tag: [binder, Android系统, 教程, github]
category: [源码分析]
excerpt: 新建aidl回调接口，用工具生成代码，并实现java/cpp服务端和测试代码
---
新建aidl回调接口，用工具生成代码，并实现java/cpp服务端和测试代码。  
基本流程和[java端代码生成](https://hqw700.github.io/2021/01/05/java-binder-gen/)，[cpp端代码生成](https://hqw700.github.io/2021/01/09/cpp-binder-gen/) 这两章差不多，都分为接口代码生成， 服务端代码，测试代码三部分。
## 一， 增加aidl回调接口
ICallback.aidl
``` java
package demo;

interface ICallback
{
    void onCallback(String str);
}
```
在ITest.aidl中增加注册回调接口
``` java
package demo;

import demo.ICallback;
interface ITest
{
    void ping();
    int sum(int x, int y);
    void registerCallback(ICallback cb);
    void unregisterCallback(ICallback cb);
}
```
## 二， JAVA端使用
### 2.1 java端回调代码生成
执行下面两条命令即可
```
$ out/host/linux-x86/bin/aidl -Iframeworks/native/cmds/binderdemo/aidl -oframeworks/native/cmds/binderdemo/java/src/ frameworks/native/cmds/binderdemo/ICallback.aidl
$ out/host/linux-x86/bin/aidl -Iframeworks/native/cmds/binderdemo/aidl -oframeworks/native/cmds/binderdemo/java/src/ frameworks/native/cmds/binderdemo/aidl/demo/ITest.aidl
$ tree 
├── aidl
│   └── demo
│       ├── ICallback.aidl
│       └── ITest.aidl
java/
├── Android.mk
├── java_binder_test
└── src
    └── demo
        ├── ICallback.java
        ├── ITest.java
        ├── Main.java
        └── TestServer.java

```
这里解释下aidl工具的用法：
**-I** 指定aidl引用的目录，因为当时有引用demo.ICallback包
**-o** 指定生成目录
其它一些没有用到，没去研究就不作解释了。

### 2.2 java服务端代码
首先需要一个远程回调的数组队列和遍历执行回调的方法 
``` java
private final RemoteCallbackList<ICallback> mCallbacks = new RemoteCallbackList<ICallback>();

protected void doCallback(String info) {
    if (mCallbacks == null || mCallbacks.getRegisteredCallbackCount() <= 0) {
        Log.d(TAG, "doCallback error");
        return;
    }
    synchronized (mCallbacks) {
        mCallbacks.beginBroadcast();
        int N = mCallbacks.getRegisteredCallbackCount();
        Log.d(TAG, "doCallback Count = " + N);
        for (int i = 0; i < N; i++) {
            try {
                if (mCallbacks.getBroadcastItem(i) == null) {
                    continue;
                }
                mCallbacks.getBroadcastItem(i).onCallback(info);
            } catch (DeadObjectException e) {
                if (mCallbacks.getBroadcastItem(i) != null)
                    mCallbacks.unregister(mCallbacks.getBroadcastItem(i));
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }
        mCallbacks.finishBroadcast();
    }
}
```

然后重写注册和反注册接口
``` java
@Override
public void registerCallback(ICallback cb) throws RemoteException {
    Log.d(TAG, "register " + cb);
    if (cb != null) mCallbacks.register(cb);
}

@Override
public void unregisterCallback(ICallback cb) throws RemoteException {
    if (cb != null) mCallbacks.unregister(cb);
}
```

最后就是回调方法的调用，我让执行ping函数后延时5秒回调
``` java
@Override
public void ping() throws RemoteException {
    Log.d(TAG, "ping receive ok, after 5s callback");
    try {
        Thread.sleep(5000);
    } catch (InterruptedException e) {}

    Log.d(TAG, "ping doCallback");
    doCallback("ping");
}
```

### 2.3 测试类Main.java
和注册死亡回调差不多, 注册之后执行ping函数，5s后会收到回调。
``` java
public static void main(String[] args) {
    ...
    testClient.registerCallback(new ICallback.Stub() {
        @Override
        public void onCallback(String str) throws RemoteException {
            Log.d(TAG_C, "This is onCallback");
        }
    });

    Log.d(TAG_C, "ping, 5s callback");
    testClient.ping();
    ...
}
```

## 三，CPP端使用
### 3.1 cpp端回调代码生成
执行下面两条指令即可
```
$ out/host/linux-x86/bin/aidl-cpp frameworks/native/cmds/binderdemo/aidl/demo/ICallback.aidl frameworks/native/cmds/binderdemo/cpp/include/ frameworks/native/cmds/binderdemo/cpp/ICallback.cpp
$ out/host/linux-x86/bin/aidl-cpp -Iframeworks/native/cmds/binderdemo/aidl frameworks/native/cmds/binderdemo/aidl/demo/ITest.aidl frameworks/native/cmds/binderdemo/cpp/include/ frameworks/native/cmds/binderdemo/cpp/ITest.cpp
$ tree
├── aidl
│   └── demo
│       ├── ICallback.aidl
│       └── ITest.aidl
├── cpp
│   ├── Android.mk
│   ├── CMakeLists.txt
│   ├── ICallback.cpp
│   ├── ITest.cpp
│   ├── include
│   │   └── demo
│   │       ├── BnCallback.h
│   │       ├── BnTest.h
│   │       ├── BpCallback.h
│   │       ├── BpTest.h
│   │       ├── ICallback.h
│   │       └── ITest.h
│   └── main.cpp
```
### 3.2 cpp服务端代码
远程回调的数组
``` cpp
std::vector<sp<demo::ICallback>> mCallbackList;
```
注册与反注册
``` cpp
binder::Status registerCallback(const sp<::demo::ICallback> &cb) override {
    for (auto& it : mCallbackList) {
        if (IInterface::asBinder(it) == IInterface::asBinder(cb)) {
            INFO("%s: Tried to add callback %p which was already subscribed",
                    __FUNCTION__, cb.get());
            return binder::Status();
        }
    }

    mCallbackList.push_back(cb);
    return binder::Status();
}

binder::Status unregisterCallback(const sp<::demo::ICallback> &cb) override {
    for (auto it = mCallbackList.begin(); it != mCallbackList.end(); it++) {
        if (IInterface::asBinder(*it) == IInterface::asBinder(cb)) {
            mCallbackList.erase(it);
            INFO("server: unregisterCallback ok");
            return binder::Status().ok();
        }
    }
    return binder::Status();
}
```

回调方法的调用，还是执行ping函数后延时5秒回调
``` cpp
for (auto& i : mCallbackList) {
    i->onCallback(String16("test"));
}
```
### 3.3 测试类main.cpp
先注册回调，然后再解除注册
``` cpp
class CallbackTest : public demo::BnCallback {
public:
    binder::Status onCallback(const String16 &str) override {
        INFO("client: onCallback, receive str: %s", String8(str).string());
        return binder::Status();
    }
};

int main(int argc, char **argv) {
    sp<ProcessState> proc(ProcessState::self());
    ProcessState::self()->startThreadPool();
    sp<IServiceManager> sm = defaultServiceManager();
    sp<IBinder> binder = sm->getService(String16(BINDER_NAME));
    sp<demo::ITest> test = interface_cast<demo::ITest>(binder);
    sp<CallbackTest> callback = new CallbackTest();
    test->registerCallback(callback);
    INFO("client: ping, 5s callback");
    test->ping();

    test->unregisterCallback(callback);
    INFO("client: ping again, 5s callback");
    test->ping();
    IPCThreadState::self()->joinThreadPool();
    return 0;
}
```
## 四， 源码
本章节源码在 https://github.com/hqw700/binderdemo/releases/tag/v0.4 下


> 操作系统：Windows 10 专业版 19042.685  
> 编译机：WSL2 Ubuntu-18.04  
> 手机：Google Pixel 1 
> 源码版本：AOSP android-7.1.1_r35