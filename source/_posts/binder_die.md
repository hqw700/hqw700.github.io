---
title: 一起来学binder — cpp/java端的死亡讣告与dump的简单示例
date: 2021-01-21 23:20:48
tag: [binder, Android系统, 教程, github]
category: [源码分析]
excerpt: 重写cpp和java端binder死亡回调与dump接口
---

> 操作系统：Windows 10 专业版 19042.685  
> 编译机：WSL2 Ubuntu-18.04  
> 手机：Google Pixel 1 
> 源码版本：AOSP android-7.1.1_r35

死亡回调是当服务端进程(Bn端)死亡时，通过回调通知到客户端进程(Bp端)。重写IBinder.DeathRecipient的binderDied函数获取回调。  
Dump是binder服务端默认功能，当终端调用``dumpsys BINDER_NAME``时，输出信息，只需要重写dump接口即可。  
所以，死亡回调是在客户端操作，dump是在服务端操作。

## java端注册死亡回调
binderdemo/java/src/demo/Main.java
``` java
    testClient.asBinder().linkToDeath(new IBinder.DeathRecipient() {
        @Override
        public void binderDied() {
            Log.d("binder", "binderDied calling~!");
            System.out.println("binderDied calling");
            System.exit(0);
        }
    }, 0);
    for (;;);
```
因为示例客户端跑完就退出了，所以最后需要加个死循环等待回调。

## cpp端的死亡回调
cpp端的回调像java那样的写法是不能工作的，因为死亡回调需要工作在binder线程池内，具体原因后面分析代码时再讲。  

``` cpp
class TestDeathRecipient : public IBinder::DeathRecipient
{
    private:
        virtual void binderDied(const wp<IBinder>& who) {
            INFO("client: binderDied");
            exit(EXIT_FAILURE);
        };
};

int main(int argc, char **argv) {
    sp<ProcessState> proc(ProcessState::self());
    ProcessState::self()->startThreadPool();
    sp<IServiceManager> sm = defaultServiceManager();
    sp<IBinder> binder = sm->getService(String16(BINDER_NAME));
    sp<TestDeathRecipient> death = new TestDeathRecipient();
    int link = binder->linkToDeath(death);
    IPCThreadState::self()->joinThreadPool();
    return 0;
}
```
## java端dump
binderdemo\java\src\demo\TestServer.java
``` java
@Override
public void dump(FileDescriptor fd, PrintWriter writer, String[] args) {
    final IndentingPrintWriter pw = new IndentingPrintWriter(writer, "  ");
    pw.println("test_server dump:");
    pw.increaseIndent();
    pw.println("this is Java implementation");
}
```
## cpp端dump
binderdemo/cpp/main.cpp#Test::dump
``` cpp
status_t dump(int fd, const Vector<String16> &args) override {
    String8 result;
    result.append("test_server dump:\n");
    result.append("this is cpp implementation\n");
    write(fd, result.string(), result.size());
    return NO_ERROR;
}
```
## 测试与源码
这个两项测试比较简单，kill掉binder服务端，客户端的binderDied就会获得回调。  
``dumpsys test_server``就会在终端输出dump的打印。  
这章主要讲用法，后续再详解死亡讣告原理  
本章节的代码在tag v0.3下：  
https://github.com/hqw700/binderdemo/releases/tag/v0.3

