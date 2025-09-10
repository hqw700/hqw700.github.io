---
title: Android13 framework单独编译
date: 2025/9/10 16:51
tag: [Android系统, 奇技淫巧]
category: [Android系统]
excerpt: Android13 framework.jar单独编译
---

> 之前一直用`make framework`单独编译framework.jar，在android13发现无效了。

## 单独编译framework（user和userdebug版本中均可使用）
```
$ make -j8 framework-minus-apex | tee
```
- 生成
```
out/target/product/xxx/system/framework/boot-ecloud-cust-client.vdex
out/target/product/xxx/system/framework/framework.jar
out/target/product/xxx/system/framework/boot-telephony-common.vdex
out/target/product/xxx/system/framework/arm64/boot-ext.oat
out/target/product/xxx/system/framework/arm64/boot-framework.oat
out/target/product/xxx/system/framework/arm64/boot-core-icu4j.oat
out/target/product/xxx/system/framework/arm64/boot-ctyun-framework.art
out/target/product/xxx/system/framework/arm64/boot-ims-common.oat
out/target/product/xxx/system/framework/arm64/boot-telephony-common.oat
out/target/product/xxx/system/framework/arm64/boot-framework-graphics.art
out/target/product/xxx/system/framework/arm64/boot-framework-graphics.oat
out/target/product/xxx/system/framework/arm64/boot-telephony-common.art
out/target/product/xxx/system/framework/arm64/boot-ctyun-framework.oat
out/target/product/xxx/system/framework/arm64/boot-core-icu4j.art
out/target/product/xxx/system/framework/arm64/boot-ext.art
out/target/product/xxx/system/framework/arm64/boot-ecloud-cust-client.oat
out/target/product/xxx/system/framework/arm64/boot-ims-common.art
out/target/product/xxx/system/framework/arm64/boot-ecloud-cust-client.art
out/target/product/xxx/system/framework/arm64/boot-framework.art
out/target/product/xxx/system/framework/arm64/boot-voip-common.art
out/target/product/xxx/system/framework/arm64/boot-voip-common.oat
out/target/product/xxx/system/framework/boot-ctyun-framework.vdex
out/target/product/xxx/system/framework/boot-framework-graphics.vdex
out/target/product/xxx/system/framework/boot-ext.vdex
out/target/product/xxx/system/framework/boot-framework.vdex
out/target/product/xxx/system/framework/boot-voip-common.vdex
out/target/product/xxx/system/framework/boot-core-icu4j.vdex
out/target/product/xxx/system/framework/boot-ims-common.vdex
out/target/product/xxx/system/etc/compatconfig/framework-platform-compat-config.xml
out/target/product/xxx/system/etc/protolog.conf.json.gz
```
- 编译耗时（i9-12900K）
```
#### build completed successfully (03:15 (mm:ss)) ####
```

## 通过adb更新（需要支持adb remount）

```
$ adb sync
```


> 源码版本：AOSP android-13.0.0_r74