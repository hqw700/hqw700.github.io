---
title: Update AOSP repo url
date: 2024/3/25 20:34
tag: [git, 教程]
category: [代码管理]
excerpt: 在已有的repo仓库下替换成其它仓库，如redroid的项目替换成aosp的项目，他们基线可能都一致，但其中个别仓库有差别。
---

## 一，复制.repo
``` shell
$ mkdir aic11
$ cp -ar redroid11/.repo aic11/
```
如果迁移到其它服务器可以先压缩或者使用rsync复制
``` shell
$ tar -zcvf redroid11.tar.gz .repo
# 或者
$ rsync -alv ${USER}@ip:~/code/.repo .
```

## 二，替换url

- 如果直接使用repo init会报错
``` shell
$ cd aic11
$ repo init --depth 1 -u https://mirrors.bfsu.edu.cn/git/AOSP/platform/manifest -b android-11.0.0_r48
repo: reusing existing repo client checkout in /mnt/data/code/aic11
error: Your local changes to the following files would be overwritten by checkout:
        default.xml
Please commit your changes or stash them before you switch branches.
Aborting
```
- 可以先删除.repo/manifest*，再进行repo init
``` shell
$ rm -rf .repo/manifest*
$ repo init --depth 1 -u https://mirrors.bfsu.edu.cn/git/AOSP/platform/manifest -b android-11.0.0_r48
```

- 然后用 --force-sync 参数进行同步，这样已经下载的仓库就不会重复下载了。
``` shell
$ repo sync --force-sync 
```


