---
title: 一起来学binder — in/out/inout以及oneway的简单示例
date: 2021-02-19 23:13:57
tag: [binder, Android系统, 教程, github]
category: [源码分析]
excerpt: binder aidl接口关键字的简单用法和区别
---

## 一，JAVA端使用
### 1.1 aidl接口
一开始写java demo的时候用Bundle来传输数据，当用cpp写的时候才发现cpp上没有bundle，cpp上可以用的PersistableBundle在java上又缺少out传输的接口，所以本章java和cpp采用不同的aidl文件，大家也可以从中发现Bundle和PersistableBundle的区别。  
增加了三个接口：sendIn/sendOut/sendInOut 分别对应binder参数in/out/inout三种模式, 没有声明的话系统默认为in模式
``` java
package demo;

import demo.ICallback;
import android.os.Bundle;
interface ITest
{
    void ping();
    oneway void pingOneway();
    void sendIn(in Bundle data);
    void sendOut(out Bundle data);
    void sendInOut(inout Bundle data);
    int sum(int x, int y);
    void registerCallback(ICallback cb);
    void unregisterCallback(ICallback cb);
}
```

### 1.2 生成代码
需要指定android.os.Bundle类的路径frameworks/base/core/java
```
out/host/linux-x86/bin/aidl -Iframeworks/native/cmds/binderdemo/aidl -Iframeworks/base/core/java -oframeworks/native/cmds/binderdemo/java/src/ frameworks/native/cmds/binderdemo/aidl/demo/ITest.aidl
```
### 1.3 客户端代码
客户在三种模式下统一发送一个Bundle对象，Bundle中包含一个"data"字段，内容为"client"。然后打印发送后该Bundle的内容。
``` java
Bundle data_in = new Bundle();
data_in.putString("data", "client");
testClient.sendIn(data_in);
Log.d(TAG_C, "after sendIn: " + data_in.getString("data"));

Bundle data_out = new Bundle();
data_out.putString("data", "client");
testClient.sendOut(data_out);
Log.d(TAG_C, "after sendOut: " + data_out.getString("data"));

Bundle data_inout = new Bundle();
data_inout.putString("data", "client");
testClient.sendInOut(data_inout);
Log.d(TAG_C, "after sendInOut: " + data_inout.getString("data"));
```

### 1.4 服务端代码
服务端统一将传入的Bundle中data字段的内容接上"server"
``` java
@Override
public void sendIn(Bundle data) throws RemoteException {
    String rec = data.getString("data");
    Log.d(TAG, "sendIn: receive: " + rec);
    data.putString("data", rec + "server");
}

@Override
public void sendOut(Bundle data) throws RemoteException {
    String rec = data.getString("data");
    Log.d(TAG, "sendOut: receive: " + rec);
    data.putString("data", rec + "server");
}

@Override
public void sendInOut(Bundle data) throws RemoteException {
    String rec = data.getString("data");
    Log.d(TAG, "sendInOut: receive: " + rec);
    data.putString("data", rec + "server");
}
```

### 1.5 测试结果 
logcat
``` log
1973-03-06 03:09:36.760 2744-2744/? D/JAVA_BINDER.Server: This is TestServer
1973-03-06 03:09:42.601 2753-2753/? D/JAVA_BINDER.Client: This is TestClient
1973-03-06 03:09:42.605 2744-2751/? D/JAVA_BINDER.Server: sendIn: receive: client
1973-03-06 03:09:42.605 2753-2753/? D/JAVA_BINDER.Client: after sendIn: client
1973-03-06 03:09:42.605 2744-2752/? D/JAVA_BINDER.Server: sendOut: receive: null
1973-03-06 03:09:42.606 2753-2753/? D/JAVA_BINDER.Client: after sendOut: nullserver
1973-03-06 03:09:42.606 2744-2751/? D/JAVA_BINDER.Server: sendInOut: receive: client
1973-03-06 03:09:42.606 2753-2753/? D/JAVA_BINDER.Client: after sendInOut: clientserver

```
![](/img/blog/binder_in_out.jpg) 

## 二，CPP端使用
### 2.1 aidl接口
由Bundle替换为PersistableBundle
``` java
package demo;

import demo.ICallback;
import android.os.PersistableBundle;
interface ITest
{
    void ping();
    oneway void pingOneway();
    void sendIn(in PersistableBundle data);
    void sendOut(out PersistableBundle data);
    void sendInOut(inout PersistableBundle data);
    int sum(int x, int y);
    void registerCallback(ICallback cb);
    void unregisterCallback(ICallback cb);
}
```
### 2.2 生成代码
```
out/host/linux-x86/bin/aidl-cpp -Iframeworks/native/cmds/binderdemo/aidl -Iframeworks/native/aidl/binder frameworks/native/cmds/binderdemo/aidl/demo/ITest.aidl frameworks/native/cmds/binderdemo/cpp/include/ frameworks/native/cmds/binderdemo/cpp/ITest.cpp
```
### 2.3 客户端代码
``` cpp
int main(int argc, char **argv) {
    ...
    os::PersistableBundle data_in;
    data_in.putString(String16("data"),String16( "client"));
    test->sendIn(data_in);
    String16 ret_in;
    data_in.getString(String16("data"), &ret_in);
    INFO("client: after sendIn: %s", String8(ret_in).string());

    os::PersistableBundle data_out;
    data_out.putString(String16("data"),String16( "client"));
    test->sendOut(&data_out);
    String16 ret_out;
    data_out.getString(String16("data"), &ret_out);
    INFO("client: after sendOut: %s", String8(ret_out).string());

    os::PersistableBundle data_inout;
    data_inout.putString(String16("data"),String16( "client"));
    test->sendInOut(&data_inout);
    String16 ret_inout;
    data_inout.getString(String16("data"), &ret_inout);
    INFO("client: after sendInOut: %s", String8(ret_inout).string());
    ...
}
```

### 2.4 服务端代码
``` cpp
class Test : public demo::BnTest {
    std::vector<sp<demo::ICallback>> mCallbackList;
public:
    binder::Status sendIn(const os::PersistableBundle &data) override {
        String16 rec;
        data.getString(String16("data"), &rec);
        INFO("server: sendIn: receive: %s", String8(rec).string());
//        rec.append(String16("server"));
//        data->putString(String16("data"), rec);
        return binder::Status();
    }

    binder::Status sendOut(os::PersistableBundle *data) override {
        String16 rec;
        data->getString(String16("data"), &rec);
        INFO("server: sendIn: receive: %s", String8(rec).string());
        rec.append(String16("server"));
        data->putString(String16("data"), rec);
        return binder::Status();
    }

    binder::Status sendInOut(os::PersistableBundle *data) override {
        String16 rec;
        data->getString(String16("data"), &rec);
        INFO("server: sendIn: receive: %s", String8(rec).string());
        rec.append(String16("server"));
        data->putString(String16("data"), rec);
        return binder::Status();
    }
};
```
### 2.5 测试结果 
测试结果和java端基本一致
``` log
1973-03-06 03:42:09.009 2818-2818/? D/cpp_binder: server: This is TestServer
1973-03-06 03:42:15.334 2821-2821/? D/cpp_binder: server: This is TestServer
1973-03-06 03:42:19.767 2823-2823/? D/cpp_binder: client: We're the client, inout test
1973-03-06 03:42:19.770 2821-2821/? D/cpp_binder: server: sendIn: receive: client
1973-03-06 03:42:19.772 2823-2823/? D/cpp_binder: client: after sendIn: client
1973-03-06 03:42:19.774 2821-2822/? D/cpp_binder: server: sendIn: receive: 
1973-03-06 03:42:19.775 2823-2823/? D/cpp_binder: client: after sendOut: server
1973-03-06 03:42:19.775 2821-2821/? D/cpp_binder: server: sendIn: receive: client
1973-03-06 03:42:19.776 2823-2823/? D/cpp_binder: client: after sendInOut: clientserver
```

## 三，总结
由测试结果可以看出：  
1， 在in(默认)模式下，会将内容传递到服务端，但服务端修改的内容不会传回来  
2， 在out模式下，不会将内容传递到服务端，但服务端修改的内容会传回来  
3， 在inout模式下，会将内容传递到服务端，服务端修改的内容也会传回来

## 四，oneway
oneway的用法比较简单，区别就是加了oneway不等待服务端执行完就返回，详细用法可以看代码中的ping()和pingOneway()的区别

## 五，源码与环境
由于java和cpp采用不同的接口，源码分成两部分。  
java部分代码在:  
https://github.com/hqw700/binderdemo/releases/tag/v0.5.1  
cpp部分代码在:  
https://github.com/hqw700/binderdemo/releases/tag/v0.5.2

> 操作系统：Windows 10 专业版 19042.685  
> 编译机：WSL2 Ubuntu-18.04  
> 手机：Google Pixel 1  
> 源码版本：AOSP android-7.1.1_r35
