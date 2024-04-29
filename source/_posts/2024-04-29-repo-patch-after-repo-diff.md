---
title: 给repo项目打patch
date: 2024/4/29 14:34
tag: [git, 教程]
category: [代码管理]
excerpt: 使用repo diff导出差异后，在其它repo项目打patch
---

## 查看当前修改，可能涉及几个不同的仓库
eg.
``` shell
$ repo status
project art/                                    (*** NO BRANCH ***)
 -m     runtime/mem_map.cc
project build/make/                             (*** NO BRANCH ***)
 -m     target/board/generic_x86/BoardConfig.mk
 -m     target/board/generic_x86/device.mk
 -m     target/product/runtime_libart.mk
project prebuilts/misc/                         (*** NO BRANCH ***)
 -m     linux-x86/flex/flex-2.5.39
project packages/apps/DeskClock/                (*** NO BRANCH ***)
 -m     src/com/android/deskclock/stopwatch/StopwatchFragment.java
project system/core/                            (*** NO BRANCH ***)
 -m     rootdir/etc/ld.config.txt
 -m     rootdir/init.rc
```

## 使用repo diff导出修改
```
repo diff > repo.diff
```

## 在其它repo仓库中导入修改
使用以下脚本
repo-diff-patch.sh
``` sh
#!/bin/sh
## Script to patch up diff reated by `repo diff`

if [ -z "$1" ] || [ ! -e "$1" ]; then
    echo "Usages: $0 <repo_diff_file>";
    exit 0;
fi

rm -fr _tmp_splits*
cat $1 | csplit -qf '' -b "_tmp_splits.%d.diff" - '/^project.*\/$/' '{*}' 

working_dir=`pwd`

for proj_diff in `ls _tmp_splits.*.diff`
do 
    chg_dir=`cat $proj_diff | grep '^project.*\/$' | cut -d " " -f 2`
    echo "FILE: $proj_diff $chg_dir"
    if [ -e $chg_dir ]; then
        ( cd $chg_dir; \
            cat $working_dir/$proj_diff | grep -v '^project.*\/$' | patch -Np1;);
    else
        echo "$0: Project directory $chg_dir don't exists.";
    fi
done
```

### 打patch
```
$ ./repo-diff-patch.sh repo.diff
FILE: _tmp_splits.0.diff
patch: **** Only garbage was found in the patch input.
FILE: _tmp_splits.1.diff art/
patching file runtime/mem_map.cc
FILE: _tmp_splits.2.diff build/make/
patching file target/board/generic_x86/BoardConfig.mk
patching file target/board/generic_x86/device.mk
patching file target/product/runtime_libart.mk
FILE: _tmp_splits.3.diff packages/apps/DeskClock/
patching file src/com/android/deskclock/stopwatch/StopwatchFragment.java
FILE: _tmp_splits.4.diff prebuilts/misc/
patching file linux-x86/flex/flex-2.5.39
FILE: _tmp_splits.5.diff system/core/
patching file rootdir/etc/ld.config.txt
patching file rootdir/init.rc
```

### After patch
其中prebuilts/misc/linux-x86/flex/flex-2.5.39为是二进制文件，patch失败
``` shell
$ repo status
project art/                                    (*** NO BRANCH ***)
 -m     runtime/mem_map.cc
project build/make/                             (*** NO BRANCH ***)
 -m     target/board/generic_x86/BoardConfig.mk
 -m     target/board/generic_x86/device.mk
 -m     target/product/runtime_libart.mk
project packages/apps/DeskClock/                (*** NO BRANCH ***)
 -m     src/com/android/deskclock/stopwatch/StopwatchFragment.java
project system/core/                            (*** NO BRANCH ***)
 -m     rootdir/etc/ld.config.txt
 -m     rootdir/init.rc
```
