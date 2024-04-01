---
title: Build android executable so lib with ndk
date: 2024/3/30 23:58
tag: [Android系统,教程,ndk]
category: [工具使用]
excerpt: 使用NDK编译可执行的so库
---

references：
[实现可执行的so动态链接库](https://www.cnblogs.com/motadou/p/9644566.html)

## Sample
```
$ tree hello-shared/
hello-shared/
├── CMakeLists.txt
├── build.sh
├── hello.c
├── main.c
```

- CMakeLists.txt
```
cmake_minimum_required(VERSION 3.5)
project (hello_shared)

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fPIC")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fPIC")

# set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,-e,lib_entry")
set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,-e__lib_main")

add_library(hello SHARED hello.c)
add_executable(hello_shared main.c)
target_link_libraries(hello_shared dl)
```

- build.sh
``` bash
#!/bin/bash
rm -rf build-android
mkdir build-android
cd build-android
cmake -DCMAKE_TOOLCHAIN_FILE=$ANDROID_NDK/build/cmake/android.toolchain.cmake \
-DANDROID_ABI="arm64-v8a" \
-DANDROID_NDK=$ANDROID_NDK \
-DANDROID_PLATFORM=android-21 ..
make
```

- hello.c
``` c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>
#include <string.h>
#include <sys/types.h>
#include <sys/stat.h>

// Must define the interpretor to be the dynamic linker
#ifdef __ANDROID__
    #ifdef __LP64__
        const char service_interp[] __attribute__((section(".interp"))) = "/system/bin/linker64";
    #else
        const char service_interp[] __attribute__((section(".interp"))) = "/system/bin/linker";
    #endif
#endif

struct CmdLine{
    int     argc;
    char ** argv;
};
 
void ReadCmdLine(struct CmdLine * pLine){
    pLine->argc = 0;
    pLine->argv = NULL;
 
    int32_t iFileId = open("/proc/self/cmdline", O_RDONLY, 0644);
    if (iFileId < 0){
        return ;
    }
 
    char szBuff[8192];
    int iFileSize = read(iFileId, szBuff, sizeof(szBuff));
    int iBegin    = 0;
    int iTemp     = 0;
    close(iFileId);
 
    for (iTemp = 0; iTemp < iFileSize; iTemp++){
        if (szBuff[iTemp] == '\0'){
            pLine->argc++;
        }
    }
 
    pLine->argv = (char **)malloc(pLine->argc * sizeof(char *));
    for (iTemp = 0, pLine->argc = 0; iTemp < iFileSize; iTemp++){
        if (szBuff[iTemp] == '\0'){
            char * p = (char *)malloc(iTemp - iBegin + 1);
            memcpy(p, szBuff + iBegin, iTemp - iBegin);
            p[iTemp - iBegin] = '\0';
            pLine->argv[pLine->argc++] = p;
            iBegin = iTemp + 1;
        }
    }
}

void __lib_main(void) {
    printf("Entry point of libhello.so \n");
 
    struct CmdLine cmdLine;
    ReadCmdLine(&cmdLine);
    printf("argc:%d\n", cmdLine.argc);
 
    int iTemp = 0;
    for ( ; iTemp < cmdLine.argc; iTemp++)
    {
        printf("arg:%s\n", cmdLine.argv[iTemp]);
    }
    _exit(0);
}

void lib_entry() {
    printf("Entry point of lib_entry\n");
    exit(0);
}

void hello() {
   printf("Entry point of hello\n");
}
```

- main.c
``` c
#include <dlfcn.h>
#include <stdio.h>

int main() {
    void *handle = dlopen("./libhello.so", RTLD_LAZY);
    if (!handle) {
        fprintf(stderr, "Error: %s\n", dlerror());
        return 1;
    }

    void (*hello)() = dlsym(handle, "hello");
    if (!hello) {
        fprintf(stderr, "Error: %s\n", dlerror());
        return 1;
    }

    hello();

    dlclose(handle);

    return 0;
}
```

## Build
``` shell
hello-shared$ ./build.sh 
-- Check for working C compiler: /home/huangqw/ndk/android-ndk-r19c/toolchains/llvm/prebuilt/linux-x86_64/bin/clang
-- Check for working C compiler: /home/huangqw/ndk/android-ndk-r19c/toolchains/llvm/prebuilt/linux-x86_64/bin/clang -- works
-- Detecting C compiler ABI info
-- Detecting C compiler ABI info - done
-- Detecting C compile features
-- Detecting C compile features - done
-- Check for working CXX compiler: /home/huangqw/ndk/android-ndk-r19c/toolchains/llvm/prebuilt/linux-x86_64/bin/clang++
-- Check for working CXX compiler: /home/huangqw/ndk/android-ndk-r19c/toolchains/llvm/prebuilt/linux-x86_64/bin/clang++ -- works
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- Configuring done
-- Generating done
-- Build files have been written to: /home/huangqw/github/hello-shared/build-android
Scanning dependencies of target hello
[ 25%] Building C object CMakeFiles/hello.dir/hello.c.o
[ 50%] Linking C shared library libhello.so
[ 50%] Built target hello
Scanning dependencies of target hello_shared
[ 75%] Building C object CMakeFiles/hello_shared.dir/main.c.o
[100%] Linking C executable hello_shared
[100%] Built target hello_shared
```

## Run in android
``` 
$ adb push build-android/libhello.so build-android/hello_shared /data/local/tmp/
$ adb shell
# cd /data/local/tmp/
# ./hello_shared
Entry point of hello

// run so lib
# ./libhello.so 111 222 333 444 555 666 777
Entry point of libhello.so
argc:8
arg:./libhello.so
arg:111
arg:222
arg:333
arg:444
arg:555
arg:666
arg:777
```