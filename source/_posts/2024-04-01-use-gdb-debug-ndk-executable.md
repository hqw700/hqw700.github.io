---
title: Use gdb debug ndk executable file
date: 2024/4/1 23:22
tag: [Android系统,教程,ndk]
category: [工具使用]
excerpt: 使用gdb调试ndk编译的可执行文件，但失败了，使用clion的gdb远程调试正常。
---

Code in previous post: [Install android ndk in linux and compile the sample code](https://hqw700.github.io/2024/03/26/2024-03-26-linux-install-android-emulator/)

## Run in adb
```
$ adb forward tcp:12345 tcp:12345
$ adb root
$ adb shell gdbserver64 :12345 /data/local/tmp/hello_cmake
Process /data/local/tmp/hello_cmake created; pid = 6923
gdbserver: Unable to determine the number of hardware watchpoints available.
gdbserver: Unable to determine the number of hardware breakpoints available.
Listening on port 12345
```

## Use gdb in comand-line is ERROR
```
$ cat << EOF > gdb.config
set sysroot /home/huangqw/github/hello-ndk
b /home/huangqw/github/hello-ndk/main.cpp:7
target remote 127.0.0.1:12345
EOF
```

Run gdb show error: **Backtrace stopped: not enough registers or memory available to unwind further**  
Can not continue
```
$ gdb-multiarch -x gdb.config
GNU gdb (Ubuntu 8.1.1-0ubuntu1) 8.1.1
Copyright (C) 2018 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word".
No symbol table is loaded.  Use the "file" command.
Make breakpoint pending on future shared library load? (y or [n]) [answered N; input not from terminal]
No executable file now.
warning: Could not load vsyscall page because no executable was specified
0x0000007fb7f3b270 in ?? ()
(gdb) bt
#0  0x0000007fb7f3b270 in ?? ()
Backtrace stopped: not enough registers or memory available to unwind further
(gdb) c
Continuing.

```

## Same operation debug in clion is OK
### clion config
![clion gdb config](/img/blog/ndk-gdb-1.png)

### clion run remote debug
![clion run remote debug](/img/blog/ndk-gdb-2.png)


