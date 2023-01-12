---
title: 【代码管理】将AOSP修改提交到自己的仓库
date: 2023/1/12 18:30
tag: [git, 教程]
category: [代码管理]
excerpt: repo manifest的使用
---
对于Android源码的管理，看到过各种各样的团队不同的骚操作：有将几百个仓库合并到一个仓库管理的，也有将几个仓库全部上传到本地的。  
我觉得正确的处理应该是像[redroid](https://github.com/remote-android/redroid-doc)一样：没涉及的修改从官方下载，有修改的创建自己的仓库。  

## 一，源码下载
官方源码通过repo命令下载
```
$ repo init https://android.googlesource.com/platform/manifest -b android-7.1.2_r33
```
通常会替换为国内的清华源https://aosp.tuna.tsinghua.edu.cn/platform/manifest, 并加上 `--depth 1` 控制git深度以节省硬盘空间。


## 二，将manifest仓库提交到自己仓库
manifest仓库管理着所有子仓库的清单，现在第一步是将这个清单提单到自己仓库，但子项目源码还是从官方下载。
### 2.1 将原origin地址修改为github地址
``` bash
$ cd .repo/manifests
.repo/manifests $ git remote remove origin
.repo/manifests $ git remote add origin git@github.com:hqw700/AospManifestTest.git
```
如果是通过`--depth 1`浅克隆的，此时push会报shallow update not allowed
``` bash
.repo/manifests $ git checkout -b android-7.1.2_r33
.repo/manifests $ git push --set-upstream origin android-7.1.2_r33
Enumerating objects: 3, done.
Counting objects: 100% (3/3), done.
Delta compression using up to 16 threads
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 6.54 KiB | 6.54 MiB/s, done.
Total 3 (delta 0), reused 0 (delta 0), pack-reused 0
To github.com:hqw700/AospManifestTest.git
 ! [remote rejected] android-7.1.2_r33 -> android-7.1.2_r33 (shallow update not allowed)
error: failed to push some refs to 'github.com:hqw700/AospManifestTest.git'
```

### 2.2 浅克隆push
浅克隆仓库的push问题，网上大多数说法是完整下载后push，但这样又带着大量不需要的提交。经过试验，可以加上`--orphan`参数抛弃所有修改重新commit。
``` bash
.repo/manifests $ git checkout --orphan android-7.1.2_r33
Switched to a new branch 'android-7.1.2_r33'
.repo/manifests $ git commit -m "Init repo"
[android-7.1.2_r33 (root-commit) 6f9b46c] Init repo
 1 file changed, 557 insertions(+)
 create mode 100644 default.xml
.repo/manifests $ git push --set-upstream origin android-7.1.2_r33
```

## 三，修改default.xml
通过上面的步骤，已经可以用`repo init git@github.com:hqw700/AospManifestTest.git -b android-7.1.2_r33`命令初始化清单了，但执行`repo sync`却发现无法下载代码，那是因为子仓库的下载地址默认是github的，但github中却无这个仓库，所以下载失败。因为需要将默认地址修改为官方的。
原default.xml的头部是这样的
```
<remote fetch=".." name="aosp" review="https://android-review.googlesource.com/" />
```
fetch=".."的意思是manifest下载地址的上一级为源地址，需要修改为`https://aosp.tuna.tsinghua.edu.cn/platform/manifest`仓库的上一级，也就是`https://aosp.tuna.tsinghua.edu.cn`
<remote fetch="https://aosp.tuna.tsinghua.edu.cn" name="aosp" review="https://android-review.googlesource.com/" />

此时就可以通过repo sync愉快的下载了。

## 四，修改某些仓库源码并提交到github
通过上面的步骤下载的代码和官方的无区别，但是要修改某些仓库代码并提交到自己仓库呢。  
可以在default.xml后面include一个自定义的xml如  
default.xml
``` xml
<include name="custom.xml" />
```
将需要删除，替换，增加的仓库放在custom.xml, 如  
custom.xml
``` xml
<manifest>
    <remote name="github" fetch="https://github.com/hqw700"/>
    <!-- <remote name="github" fetch="git@github.com"/> -->

    <!-- 删除原代码仓库 -->
    <remove-project name="platform/prebuilts/qemu-kernel"/>
    <!-- 替换为自己的仓库示例，未创建该仓库 -->
    <!-- <project path="prebuilts/qemu-kernel" name="prebuilts/qemu-kernel" remote="github" revision="master"/> -->

    <!-- 新增仓库示例 -->
    <project path="frameworks/native/cmds/binderdemo" name="binderdemo" remote="github" revision="main"/>
</manifest>
```

### 4.1 project参数
name 为下载仓库地址，结构为remote + name  
path为本地目录地址  
每个project仓库后面可以指定远程仓库地址remote="xxx", 以及git commint id或者分支名revision="xxx", 也可以指定git深度减少体积clone-depth="1"  
另外，将多个manifest放在同一个仓库下不冲突，如果AOSP代码和emulator代码，在repo init时指定分支就行。

## 五，测试
完整步骤如上，如果有描述不清的地方可以看仓库的提交记录。可以注释掉大部分project来测试效果。
```
$ repo init git@github.com:hqw700/AospManifestTest.git -b android-7.1.2_r33
$ repo sync

Fetching: 100% (2/2), done in 2.510s
repo sync has finished successfully.
```