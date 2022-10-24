---
title: 【一起来学binder】最简单的binder调用
date: 2022-10-24 16:22:56
tag: [binder, Android系统, 教程, github]
category: [源码分析]
excerpt: 最简单的binder调用代码示例
---

Android提供简单的binder服务测试命令service([代码](https://cs.android.com/android/platform/superproject/+/master:frameworks/native/cmds/service/service.cpp)), 比如说要打开画面更新闪烁，可以发送 
``` text
adb shell service call SurfaceFlinger 1002 i32 1
```
其中，SurfaceFlinger是服务名称，1002是需要调用的函数id, i32 1 表示写一个32-bit integer为1的参数。  
如果要调用binderdemo里的ping方法:  
``` text
adb shell service call test_server 1
```

## CPP的最简单调用
下面是提取了service.cpp的代码写的最简短的ping调用：
``` cpp
#define LOG_TAG "simplest_call"

#include <stdlib.h>

#include <utils/RefBase.h>
#include <utils/Log.h>
#include <binder/TextOutput.h>
#include <binder/IInterface.h>
#include <binder/IBinder.h>
#include <binder/ProcessState.h>
#include <binder/IServiceManager.h>
#include <binder/IPCThreadState.h>

using namespace android;

// get the name of the generic interface we hold a reference to
static String16 get_interface_name(sp<IBinder> service)
{
    if (service != nullptr) {
        Parcel data, reply;
        status_t err = service->transact(IBinder::INTERFACE_TRANSACTION, data, &reply);
        if (err == NO_ERROR) {
            return reply.readString16();
        }
    }
    return String16();
}

int main(int argc, char* const argv[]) {
    sp<IServiceManager> sm = defaultServiceManager();
    sp<IBinder> service = sm->checkService(String16("test_server"));
    // Not necessary function, can be replaced with ifName = "demo.ITest"
    String16 ifName = get_interface_name(service);

    if (service != nullptr && ifName.size() > 0) {
        Parcel data, reply;
        // the interface name is first
        data.writeInterfaceToken(ifName);
        // ping
        service->transact(IBinder::FIRST_CALL_TRANSACTION + 0, data, &reply);
        aout << "Result: " << reply << endl;
    }
}
```

## JAVA的最简单调用
``` java
package simplestcall;

import android.os.Bundle;
import android.os.IBinder;
import android.os.Parcel;
import android.os.RemoteException;
import android.os.ServiceManager;
import android.util.Log;

public class Main {
    private static final String TAG = "SimplestCall";

    public static void main(String[] args) {
        IBinder service = ServiceManager.getService("test_server");
        final Parcel data = Parcel.obtain();
        final Parcel reply = Parcel.obtain();
        data.writeInterfaceToken("demo.ITest");
        try {
            service.transact(IBinder.FIRST_CALL_TRANSACTION + 0, data, reply, 0 /* flags */);
        } catch (RemoteException e) {
            e.printStackTrace();
        }
        System.out.println(reply.toString());
        reply.recycle();
        data.recycle();
    }
}
```

## 完整代码
完整代码在tag 0.6下的simplestcall文件夹  
https://github.com/hqw700/binderdemo/releases/tag/v0.6

