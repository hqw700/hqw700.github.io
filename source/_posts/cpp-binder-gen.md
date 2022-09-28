---
title: 【一起来学binder】cpp端代码生成
date: 2021-01-09 12:29:26
tag: [binder, Android系统, 教程, github]
category: [源码分析]
excerpt: 通过系统自带的aidl-cpp工具生成binder类，并编写客户端和服务端代码测试
---

> 操作系统：Windows 10 专业版 19042.685  
> 编译机：WSL2 Ubuntu-18.04  
> 手机：Google Pixel 1 
> 源码版本：AOSP android-7.1.1_r35

## 1，创建aidl文件
ITest.aidl
``` java
package demo;

interface ITest
{
    void ping();
    int sum(int x, int y);
}
```

## 2，用aidl-cpp工具生成cpp端接口实现
在aosp源码下载操作，详细生成结果见Github: https://github.com/hqw700/binderdemo/releases/tag/v0.2 
``` console
$ mkdir -p frameworks/native/cmds/binderdemo/cpp/include
$ out/host/linux-x86/bin/aidl-cpp frameworks/native/cmds/binderdemo/ITest.aidl frameworks/native/cmds/binderdemo/cpp/include/ frameworks/native/cmds/binderdemo/cpp/ITest.cpp
$ tree frameworks/native/cmds/binderdemo/cpp
frameworks/native/cmds/binderdemo/cpp
├── ITest.cpp
└── include
    └── demo
        ├── BnTest.h
        ├── BpTest.h
        └── ITest.h
```

## 3，binder服务端代码
和测试代码main.cpp在同一文件下
``` cpp 
class Test : public demo::BnTest {
public:
    binder::Status ping() override {
        INFO("ping receive ok");
        return binder::Status();
    }

    binder::Status sum(int32_t v1, int32_t v2, int32_t *_aidl_return) override {
        INFO("sum: %d + %d", v1, v2);
        *_aidl_return = v1 + v2;
        return binder::Status();
    }
};
```

## 4，测试类main.cpp
``` cpp 
#define LOG_TAG "cpp_binder"

#include <stdlib.h>

#include <utils/RefBase.h>
#include <utils/Log.h>
#include <binder/TextOutput.h>

#include <binder/IInterface.h>
#include <binder/IBinder.h>
#include <binder/ProcessState.h>
#include <binder/IServiceManager.h>
#include <binder/IPCThreadState.h>
#include <demo/ITest.h>
#include <demo/BnTest.h>

#define BINDER_NAME "test_server"

using namespace android;

#define INFO(...) \
    do { \
        printf(__VA_ARGS__); \
        printf("\n"); \
        ALOGD(__VA_ARGS__); \
    } while(0)

void assert_fail(const char *file, int line, const char *func, const char *expr) {
    INFO("assertion failed at file %s, line %d, function %s:",
         file, line, func);
    INFO("%s", expr);
    abort();
}

#define ASSERT(e) \
    do { \
        if (!(e)) \
            assert_fail(__FILE__, __LINE__, __func__, #e); \
    } while(0)

// ################# For binder server start #####################
class Test : public demo::BnTest {
public:
    binder::Status ping() override {
        INFO("server: ping receive ok");
        return binder::Status();
    }

    binder::Status sum(int32_t v1, int32_t v2, int32_t *_aidl_return) override {
        INFO("server: sum: %d + %d", v1, v2);
        *_aidl_return = v1 + v2;
        return binder::Status();
    }
};
// ################# For binder server end #####################

int main(int argc, char **argv) {
    if (argc == 1) {
        defaultServiceManager()->addService(String16(BINDER_NAME), new Test());
        ProcessState::self()->startThreadPool();
        INFO("server: This is TestServer");
        IPCThreadState::self()->joinThreadPool();
    } else if (argc == 3) {
        INFO("client: We're the client");
        sp<IServiceManager> sm = defaultServiceManager();
        ASSERT(sm != 0);
        sp<IBinder> binder = sm->getService(String16(BINDER_NAME));
        ASSERT(binder != 0);
        sp<demo::ITest> test = interface_cast<demo::ITest>(binder);
        ASSERT(test != 0);
        INFO("client: ping");
        test->ping();
        int ret = 0;
        test->sum(atoi(argv[1]), atoi(argv[2]), &ret);
        INFO("client: We're the client: test->sum return: %d", ret);
    }
    return 0;
}
```

## 5，Android.mk 与 CMakeLists.txt 文件 
Android.mk 用于系统编译，CMakeLists.txt 文件用于clion索引，顺便用clion将每个文件重新format了。
``` makefile
LOCAL_PATH:= $(call my-dir)
include $(CLEAR_VARS)

LOCAL_SRC_FILES:= \
    main.cpp \
    ITest.cpp

LOCAL_C_INCLUDES := $(LOCAL_PATH)/include

LOCAL_CFLAGS += -Wall

LOCAL_SHARED_LIBRARIES := \
    libcutils \
    liblog \
    libandroidfw \
    libutils \
    libbinder

LOCAL_MODULE:= cpp_binder_test

include $(BUILD_EXECUTABLE)
```
## 6，编译并push进系统
编译
``` console
$ mmm frameworks/native/cmds/binderdemo/cpp/
[100% 7/7] Install: out/target/product/sailfish/system/bin/cpp_binder_test
make: Leaving directory '/home/huangqw/code/aosp'

#### make completed successfully (1 seconds) ####
```
push
``` console
$ adb root && adb remount
$ adb push cpp_binder_test  /system/bin/
```

## 7，测试
启动服务端
``` console
$ adb shell
sailfish:/ # cpp_binder_test
server: This is TestServer

```
查看Binder服务：
第一个就是了
``` console
$ adb shell service list | grep test_
0       test_server: [demo.ITest]
```

客户端测试：
``` console
$ adb shell
sailfish:/ # cpp_binder_test 1 2
client: We're the client
client: ping
client: We're the client: test->sum return: 3
```
和上一章java端的应用和可以通信
``` console
sailfish:/ # java_binder_test 2 3
5
```
log
``` log
1973-02-06 15:13:29.710 2648-2648/? D/cpp_binder: server: This is TestServer
1973-02-06 15:13:33.266 2651-2651/? D/cpp_binder: client: We're the client
1973-02-06 15:13:33.270 2651-2651/? D/cpp_binder: client: ping
1973-02-06 15:13:33.270 2648-2649/? D/cpp_binder: server: ping receive ok
1973-02-06 15:13:33.271 2648-2648/? D/cpp_binder: server: sum: 1 + 2
1973-02-06 15:13:33.272 2651-2651/? D/cpp_binder: client: We're the client: test->sum return: 3

1973-02-06 15:38:07.240 2783-2783/? D/JAVA_BINDER.Client: This is TestClient
1973-02-06 15:38:07.242 2783-2783/? D/JAVA_BINDER.Client: ping
1973-02-06 15:38:07.243 2648-2650/? D/cpp_binder: server: ping receive ok
1973-02-06 15:38:07.243 2783-2783/? D/JAVA_BINDER.Client: sum (2,3)
1973-02-06 15:38:07.243 2648-2649/? D/cpp_binder: server: sum: 2 + 3
1973-02-06 15:38:07.244 2783-2783/? D/JAVA_BINDER.Client: sum (2,3) return 5
```

## 8，源码
这个binder系列会持续更新，逐渐完善更多的用法，本章节的代码在tag v0.2下： 
https://github.com/hqw700/binderdemo/releases/tag/v0.2