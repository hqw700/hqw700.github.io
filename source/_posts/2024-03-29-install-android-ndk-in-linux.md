---
title: Install android ndk in linux and compile the sample code
date: 2024/3/29 22:55
tag: [Android系统,教程]
category: [工具使用]
excerpt: 在Linux安装ndk并编译示例代码。
---

## NDK install
``` shell
$ mkdir -p ~/ndk
$ cd ~/ndk/
$ curl -O https://dl.google.com/android/repository/android-ndk-r19c-linux-x86_64.zip
$ unzip android-ndk-r19c-linux-x86_64.zip
$ export ANDROID_NDK=/home/test/ndk/android-ndk-r19c
```

## Sample

``` shell
$ mkdir hello-ndk
$ cd hello-ndk/
```
- main.cpp
``` cpp
$ cat << EOF > main.cpp
#include <iostream>

int main(int argc, char *argv[])
{
   std::cout << "Hello ndk!" << std::endl;
   return 0;
}
EOF
```
- CMakeLists.txt
```
$ cat  << EOF > CMakeLists.txt
cmake_minimum_required(VERSION 3.5)
project (hello_cmake)

add_executable(hello_cmake main.cpp)
EOF
```

```
$ tree
.
├── CMakeLists.txt
└── main.cpp
```

## Build for linux executable
``` shell
$ mkdir build-linux
$ cd build-linux
$ cmake ..
-- The C compiler identification is GNU 7.5.0
-- The CXX compiler identification is GNU 7.5.0
-- Check for working C compiler: /usr/bin/cc
-- Check for working C compiler: /usr/bin/cc -- works
-- Detecting C compiler ABI info
-- Detecting C compiler ABI info - done
-- Detecting C compile features
-- Detecting C compile features - done
-- Check for working CXX compiler: /usr/bin/c++
-- Check for working CXX compiler: /usr/bin/c++ -- works
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- Configuring done
-- Generating done
-- Build files have been written to: /home/test/github/hello-ndk/build-linux
$ make
Scanning dependencies of target hello_cmake
[ 50%] Building CXX object CMakeFiles/hello_cmake.dir/main.cpp.o
[100%] Linking CXX executable hello_cmake
[100%] Built target hello_cmake
$ ./hello_cmake
Hello ndk!
```

## Build for android executable
``` shell
$ mkdir build-android
$ cd build-android
$ cmake -DCMAKE_TOOLCHAIN_FILE=$ANDROID_NDK/build/cmake/android.toolchain.cmake -DANDROID_ABI="arm64-v8a" -DANDROID_NDK=$ANDROID_NDK -DANDROID_PLATFORM=android-21 ..
-- Check for working C compiler: /home/test/ndk/android-ndk-r19c/toolchains/llvm/prebuilt/linux-x86_64/bin/clang
-- Check for working C compiler: /home/test/ndk/android-ndk-r19c/toolchains/llvm/prebuilt/linux-x86_64/bin/clang -- works
-- Detecting C compiler ABI info
-- Detecting C compiler ABI info - done
-- Detecting C compile features
-- Detecting C compile features - done
-- Check for working CXX compiler: /home/test/ndk/android-ndk-r19c/toolchains/llvm/prebuilt/linux-x86_64/bin/clang++
-- Check for working CXX compiler: /home/test/ndk/android-ndk-r19c/toolchains/llvm/prebuilt/linux-x86_64/bin/clang++ -- works
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- Configuring done
-- Generating done
-- Build files have been written to: /home/test/github/hello-ndk/build-android

$ make 
Scanning dependencies of target hello_cmake
[ 50%] Building CXX object CMakeFiles/hello_cmake.dir/main.cpp.o
[100%] Linking CXX executable hello_cmake
[100%] Built target hello_cmake
$ ./hello_cmake
-bash: ./hello_cmake: cannot execute binary file: Exec format error
```
### Run in android
``` shell
$ adb push build-android/hello_cmake /data/local/tmp
# cd /data/local/tmp/
/data/local/tmp # chmod +x hello_cmake ; ./hello_cmake
Hello ndk!
/data/local/tmp # file hello_cmake
hello_cmake: ELF shared object, 64-bit LSB arm64, dynamic (/system/bin/linker64)
```

## Diff
### file 
``` shell
$ file build-linux/hello_cmake
build-linux/hello_cmake: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 3.2.0, BuildID[sha1]=8e568fb9403912403a9daeb3a85cc58b8dad46f7, not stripped

$ file build-android/hello_cmake
build-android/hello_cmake: ELF 64-bit LSB shared object, ARM aarch64, version 1 (SYSV), dynamically linked, interpreter /system/bin/linker64, BuildID[sha1]=322bed0b22b9c27619b3b3a128322993667a5e7a, with debug_info, not stripped
```
``` shell
$ readelf -h build-linux/hello_cmake
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              DYN (Shared object file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x7b0
  Start of program headers:          64 (bytes into file)
  Start of section headers:          7064 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         9
  Size of section headers:           64 (bytes)
  Number of section headers:         29
  Section header string table index: 28
```

### readelf -h
``` shell
$ readelf -h build-android/hello_cmake
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              DYN (Shared object file)
  Machine:                           AArch64
  Version:                           0x1
  Entry point address:               0xdf84
  Start of program headers:          64 (bytes into file)
  Start of section headers:          5142376 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         9
  Size of section headers:           64 (bytes)
  Number of section headers:         39
  Section header string table index: 36
```