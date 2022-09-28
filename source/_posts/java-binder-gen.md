---
title: 【一起来学binder】java端代码生成
date: 2021-01-05 22:54:10
tag: [binder, Android系统, 教程, github]
category: [源码分析]
excerpt: 通过系统自带的aidl工具生成binder类，并编写客户端和服务端代码测试
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
## 2，用aidl工具生成java端接口实现
在aosp源码下载操作，然后第一步就生成错误了
``` log
~/code/aosp$ out/host/linux-x86/bin/aidl -oframeworks/native/cmds/bindergen/java/src/ frameworks/native/cmds/bindergen/ITest.aidl
frameworks/native/cmds/bindergen/ITest.aidl:3 interface ITest should be declared in a file called demo/ITest.aidl.
aidl E  1991  1991 aidl.cpp:545] Invalid package declaration 'demo'
```
原来是要在包含包名"demo"的文件夹下生成，只能将原定的文件夹名bindergen改为binderdemo。aidl工具应该是要在完整包路径下执行的，但为了目录结构好看，我放在了根目录。继续执行：
``` console
out/host/linux-x86/bin/aidl -oframeworks/native/cmds/binderdemo/java/src/ frameworks/native/cmds/binderdemo/ITest.aidl
```
无输出，不过可以看到src目录下多了demo/ITest.java文件

## 3，编写binder服务端代码
java/src/demo/TestServer.java
``` java
package demo;

import android.os.RemoteException;
import android.util.Log;

public class TestServer extends ITest.Stub {
    private static final String TAG = "JAVA_BINDER.Server";
    @Override
    public void ping() throws RemoteException {
        Log.d(TAG, "ping receive ok");
    }

    @Override
    public int sum(int x, int y) throws RemoteException {
        Log.d(TAG, "sum " + x + " + " + y);
        return x + y;
    }
}
```
## 4，编写测试类Main.java
将binder客户端和服务端合在一起，带参数执行为客户端，不带参数执行为服务端。
java/src/demo/Main.java
``` java
package demo;

import android.os.RemoteException;
import android.os.ServiceManager;
import android.util.Log;

public class Main {
    private static final String TAG_S = "JAVA_BINDER.Server";
    private static final String TAG_C = "JAVA_BINDER.Client";
    private static final String BINDER_NAME = "test_server";

    public static void main(String[] args) {
        int len = args.length;
        if (len == 0) {
            Log.d(TAG_S, "This is TestServer");
            TestServer testServer = new TestServer();
            ServiceManager.addService(BINDER_NAME, testServer);
            for (;;);
        } else {
            Log.d(TAG_C, "This is TestClient");
            ITest testClient = ITest.Stub.asInterface(ServiceManager.getService(BINDER_NAME));
            if (testClient == null) {
                System.err.println("TestServer is null");
                System.exit(-1);
            }
            try {
                Log.d(TAG_C, "ping");
                testClient.ping();

                int v1 = Integer.parseInt(args[0]);
                int v2 = Integer.parseInt(args[1]);
                Log.d(TAG_C, String.format("sum (%d,%d)", v1, v2));
                int ret = testClient.sum(v1, v2);
                Log.d(TAG_C, String.format("sum (%d,%d) return %d", v1, v2, ret));
                System.out.println(ret);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
            System.exit(0);
        }
    }
}
```

## 5，编写可以执行文件java_binder_test和Android.mk
java/java_binder_test
``` shell
# shell.
# -agentlib:jdwp=transport=dt_socket,suspend=y,server=y,address=5005
base=/system
export CLASSPATH=$base/framework/java_binder_test.jar
exec app_process $base/bin demo.Main "$@"
```

java/Android.mk
``` Makefile
LOCAL_PATH:= $(call my-dir)

include $(CLEAR_VARS)
LOCAL_SRC_FILES := $(call all-subdir-java-files)
LOCAL_MODULE := java_binder_test
include $(BUILD_JAVA_LIBRARY)

include $(CLEAR_VARS)
LOCAL_MODULE := java_binder_test
LOCAL_SRC_FILES := java_binder_test
LOCAL_MODULE_CLASS := EXECUTABLES
LOCAL_MODULE_TAGS := optional
include $(BUILD_PREBUILT)
```

## 6，编译
``` log
$ cd binderdemo/java
$ mm | tee
[ 12% 1/8] target Prebuilt: java_binder_test (out/target/product/sailfish/obj/EXECUTABLES/java_binder_test_intermediates/java_binder_test)
[ 25% 2/8] Install: out/target/product/sailfish/system/bin/java_binder_test
[ 37% 3/8] Ensure Jack server is installed and started
Jack server already installed in "/home/huangqw/.jack-server"
Launching Jack server java -XX:MaxJavaStackTraceDepth=-1 -Djava.io.tmpdir=/tmp -Dfile.encoding=UTF-8 -XX:+TieredCompilation -cp /home/huangqw/.jack-server/launcher.jar com.android.jack.launcher.ServerLauncher
[ 50% 4/8] Building with Jack: out/target/common/obj/JAVA_LIBRARIES/java_binder_test_intermediates/with-local/classes.dex
[ 62% 5/8] Copying: out/target/common/obj/JAVA_LIBRARIES/java_binder_test_intermediates/classes.dex
[ 75% 6/8] target Jar: java_binder_test (out/target/common/obj/JAVA_LIBRARIES/java_binder_test_intermediates/javalib.jar)
[ 87% 7/8] build out/target/product/sailfish/obj/JAVA_LIBRARIES/java_binder_test_intermediates/javalib.jar
[100% 8/8] Install: out/target/product/sailfish/system/framework/java_binder_test.jar
```
生成 /system/bin/java_binder_test 和 /system/framework/java_binder_test.jar

## 7，push进系统
``` console
$ adb root && adb remount
$ adb push /system/framework/java_binder_test.jar /system/framework/java_binder_test.jar
$ adb push /system/bin/java_binder_test /system/bin/java_binder_test
$ adb reboot
```
## 8，测试
启动服务端
``` console
$ adb root
$ adb shell java_binder_test

```

查看Binder服务：
``` console
$ adb shell service list | grep test_
0       test_server: [demo.ITest]
```
客户端测试：
``` console
$ adb shell java_binder_test 1 2
3
```
log
``` log
1973-02-02 03:03:23.423 2639-2639/? D/JAVA_BINDER.Server: This is TestServer
1973-02-02 03:04:37.269 2652-2652/? D/JAVA_BINDER.Client: This is TestClient
1973-02-02 03:04:37.270 2652-2652/? D/JAVA_BINDER.Client: ping
1973-02-02 03:04:37.270 2639-2647/? D/JAVA_BINDER.Server: ping receive ok
1973-02-02 03:04:37.271 2652-2652/? D/JAVA_BINDER.Client: sum (1,2)
1973-02-02 03:04:37.271 2639-2648/? D/JAVA_BINDER.Server: sum 1 + 2
1973-02-02 03:04:37.271 2652-2652/? D/JAVA_BINDER.Client: sum (1,2) return 3
```
## 9，源码
这个binder系列会持续更新，逐渐完善更多的用法，本章节的代码在tag v0.1下：  
https://github.com/hqw700/binderdemo/releases/tag/v0.1
```
binderdemo
├── ITest.aidl
└── java
    ├── Android.mk
    ├── java_binder_test
    └── src
        └── demo
            ├── ITest.java
            ├── Main.java
            └── TestServer.java
```

