---
title: Build Android emulator source code
date: 2024/3/28 20:23
tag: [虚拟化,Android系统,教程]
category: [虚拟化]
excerpt: 编译Android emulator模拟器源码
---

## Source code download
``` shell
$ sudo curl https://mirrors.bfsu.edu.cn/git/git-repo -o /usr/local/bin/repo
$ sudo chmod +x /usr/local/bin/repo
$ export REPO_URL='https://mirrors.bfsu.edu.cn/git/git-repo'
$ sudo ln /usr/bin/python3 /usr/bin/python
$ repo init --depth 1 https://mirrors.bfsu.edu.cn/git/AOSP/platform/manifest -b emu-30-release
$ repo sync -c --no-tags
```

## Build
``` shell
$ ./android/rebuild.sh  --no-tests --config debug
...
Debug build enabled.
ASAN may be in use; recommend disabling some ASAN checks as build is not
universally ASANified. This can be done with
. android/envsetup.sh

or export ASAN_OPTIONS=detect_leaks=0:detect_container_overflow=0:detect_odr_violation=0:symbolize=1
```

## Run
Run as previous post: [Install Android SDK and run emulator in linux](https://hqw700.github.io/2024/03/26/2024-03-26-linux-install-android-emulator/)
``` shell
$ export ASAN_OPTIONS=detect_leaks=0:detect_container_overflow=0:detect_odr_violation=0:symbolize=1
$ export ANDROID_SDK_ROOT=/home/test/sdk
$ export PATH=$ANDROID_SDK_ROOT/cmdline-tools/latest/bin:$ANDROID_SDK_ROOT/platform-tools:$PATH
$ objs/distribution/emulator/emulator -avd test -shell-serial stdio -allow-host-audio -skip-adb-auth -no-snapshot -no-window -cores 4 -writable-system -partition-size 8192 -gpu host -qemu -cpu host -enable-kvm -m 4096
```