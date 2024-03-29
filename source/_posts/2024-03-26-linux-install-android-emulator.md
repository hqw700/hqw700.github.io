---
title: Install Android SDK and run emulator in linux
date: 2024/3/26 23:18
tag: [虚拟化,教程,环境搭建]
category: [虚拟化]
excerpt: 在linux下载并运行Android emulator虚拟机
---

## Install related dependencies
- centos 
sudo yum install java-11-openjdk

- ubuntu
sudo apt install openjdk-11-jdk


## Download Android SDK
``` sh
#!/bin/bash
mkdir -p ~/sdk/cmdline-tools
cd ~/sdk/cmdline-tools
wget https://dl.google.com/android/repository/commandlinetools-linux-8512546_latest.zip
unzip commandlinetools-linux-8512546_latest.zip
mv cmdline-tools/ latest
cd ..
echo y | ./cmdline-tools/latest/bin/sdkmanager emulator 
echo y | ./cmdline-tools/latest/bin/sdkmanager "platform-tools" "platforms;android-28"
echo y | ./cmdline-tools/latest/bin/sdkmanager "system-images;android-28;default;x86_64"
```

## Source env
```
export ANDROID_SDK_ROOT=/home/test/sdk
export PATH=$ANDROID_SDK_ROOT/cmdline-tools/latest/bin:$ANDROID_SDK_ROOT/platform-tools:$PATH
```

## Create and run emulator avd
- Create avd
`avdmanager create avd -n test -k "system-images;android-28;default;x86_64" -d "Galaxy Nexus"`
- Run avd
`./emulator/emulator -avd test -shell-serial stdio -allow-host-audio -skip-adb-auth -no-snapshot -no-window -cores 4 -writable-system -partition-size 8192 -gpu host -qemu -cpu host  -enable-kvm -m 4096`

## Run adb
``` shell
$ adb devices
List of devices attached
emualtor-5554        device
```

## Troubleshooting
- ERROR
``` shell
INFO    | Android emulator version 31.3.13.0 (build_id 9189900) (CL:N/A)
emulator: INFO: Found systemPath /home/test/sdk/./system-images/android-28/default/x86_64/
INFO    | Duplicate loglines will be removed, if you wish to see each indiviudal line launch with the -log-nofilter flag.
WARNING | System image is writable
ProbeKVM: This user doesn't have permissions to use KVM (/dev/kvm).
The KVM line in /etc/group is: [LINE_NOT_FOUND]

If the current user has KVM permissions,
the KVM line in /etc/group should end with ":" followed by your username.

If we see LINE_NOT_FOUND, the kvm group may need to be created along with permissions:
    sudo groupadd -r kvm
    # Then ensure /lib/udev/rules.d/50-udev-default.rules contains something like:
    # KERNEL=="kvm", GROUP="kvm", MODE="0660"
    # and then run:
    sudo gpasswd -a $USER kvm

If we see kvm:... but no username at the end, running the following command may allow KVM access:
    sudo gpasswd -a $USER kvm

You may need to log out and back in for changes to take effect.

ERROR   | x86_64 emulation currently requires hardware acceleration!
CPU acceleration status: This user doesn't have permissions to use KVM (/dev/kvm).
The KVM line in /etc/group is: [LINE_NOT_FOUND]

If the current user has KVM permissions,
the KVM line in /etc/group should end with ":" followed by your username.

If we see LINE_NOT_FOUND, the kv
More info on configuring VM acceleration on Linux:
https://developer.android.com/studio/run/emulator-acceleration#vm-linux
General information on acceleration: https://developer.android.com/studio/run/emulator-acceleration.
```

### Fixed
``` shell
Permissions on /dev/kvm is currently (incorrect group and permissions):

$ ls -l /dev/kvm
crw------- 1 root root 10, 232 Jul  5 13:27 /dev/kvm
This should ideally be:
crw-rw---- 1 root kvm 10, 232 Jul  5 13:27 /dev/kvm

Manually changing it helps boot up the emulator. However, the settings are reverted on restart.

$ sudo chgrp kvm /dev/kvm
$ sudo chmod g+rw /dev/kvm
```