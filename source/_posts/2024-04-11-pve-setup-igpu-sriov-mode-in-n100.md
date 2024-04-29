---
title: PVE setup igpu sriov mode in N100
date: 2024/4/11 22:53
tag: [虚拟化,PVE,教程]
category: [虚拟化]
excerpt: 在Intel n100的PVE下设置SRIOV核显切分
---

previous post: [PVE setup in n100](https://hqw700.github.io/2024/04/10/2024-04-10-pve-setup-in-n100/)
## grub 配置
edit /etc/default/grub
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on i915.enable_guc=3 i915.max_vfs=7"
```
After edit
```
update-grub
update-initramfs -u
reboot
```

## modules 配置
edit /etc/modules
```
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
```

## 安装linux headers
```
wget http://mirrors.ustc.edu.cn/proxmox/debian/dists/bookworm/pve-no-subscription/binary-amd64/proxmox-headers-6.5.11-8-pve_6.5.11-8_amd64.deb
dpkg -i proxmox-headers-6.5.11-8-pve_6.5.11-8_amd64.deb
```

## 安装依赖包
```
apt install -y  build-* git unzip dkms
```

## 下载并编译sriov驱动
### 代码下载
``` 
cd /usr/src
git clone https://github.com/strongtz/i915-sriov-dkms i915-sriov-dkms-6.5
```

## 代码修改
edit /usr/src/i915-sriov-dkms-6.1/dkms.conf with the following:
```
PACKAGE_NAME="i915-sriov-dkms"
PACKAGE_VERSION="6.5"
```

## 编译
``` 
dkms install --force -m i915-sriov-dkms -v 6.5
reboot
```
![](/img/blog/pve-sriov-1.png)

## 重启后查看dmesg
``` 
dmesg | grep -ie guc -ie hc -ie sriov
```
显示 **Running in SR-IOV PF mode**，说明驱动加载成功
![](/img/blog/pve-sriov-2.png)

## 设置核显切分
```
echo 7 > /sys/devices/pci0000:00/0000:00:02.0/sriov_numvfs
```
`lspci | grep -i vga` 显示有8个显卡，说明切分成功
![](/img/blog/pve-sriov-3.png)
