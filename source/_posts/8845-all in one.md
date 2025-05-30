---
abbrlink: ''
categories: []
date: '2025-04-01T19:48:10.348923+08:00'
tags:
- pve
title: 8845-all in one
updated: '2025-05-10T20:08:34.218+08:00'
---
其他X86多网口设备均可以参考

# 方案一

PVE做底层，其他全部用虚拟机解决

直通核显屡次尝试失败，要么开始就不能用，要么用一会就不行了

要么直通成功，没问题了，但是测试虚拟机里安装rocm，无法使用780m核显，且vram占用一直为0

# 方案二

PVE直通核显失败

于是我用fedora做实体机，然后虚拟机里运行PVE

需要gpu的服务都放fedora实体机上

不需要的都放虚拟机里的PVE，这样即使重装也只需要恢复使用gpu的服务

我需要使用gpu的服务有comfyui, 影视（jellyfin或者emby等选一个，其实用的也不多，基本上看alist就行了）

## 第一步

### 创建网桥

必须双网口及以上才行

```bash
# 我的两个网口分别是 enp2s0 enp3s0    使用enp2s0作为网桥   切换到root设置更方便
nmcli connection add type bridge-slave con-name pve-port1 ifname enp2s0 master Pve
# pve-port1和Pve可以自定义,   enp2s0是你需要作为网桥的设备名称
nmcli connection modify pve ipv4.addresses 192.168.31.43/24
# 192.168.31.43/24  网桥的ip
nmcli connection modify pve ipv4.gateway 192.168.31.1
# 网桥的网关
nmcli connection modify pve ipv4.dns "8.8.8.8"
# 网桥的DNS
nmcli connection modify pve ipv4.method manual
# 切换网桥为手动配置
nmcli connection up pve
# 启动网桥
firewall-cmd --add-interface=pve --zone=trusted --permanent
# 无条件放行网桥流量
```

## 第二步

使用fedora的libvirt创建PVE虚拟机

```bash
dnf install @virtualization
```

可以使用virt-manager以gui方式创建虚拟机

**网络源选桥接设备      设备名称填网桥名称**

## 第三步

直通需要的设备

pci设备直通到PVE虚拟机后，无法再直通到PVE下的虚拟机，因为PVE无法开启iommu

但我要直通的pci设备只有m2转sata的转接卡

只需要把它直通给PVE就行了

然后通过`scsi`把硬盘块直通给虚拟机就行

```bash
scsi1: /dev/sdb,cache=directsync,discard=on,iothread=1
scsi2: /dev/sda,cache=directsync,discard=on,iothread=1
# 更多参数询问ai
```

至此，all in one完成，后续需要加服务，不需要gpu添加到虚拟机PVE

需要GPU的直接添加到宿主机，把宿主机服务的部署整理成脚本，重置后一键恢复服务

GPU失败是因为驱动问题

解决驱动问题

[卸载vfio](https://blog.akkia.moe/note/PVE%EF%BC%9A%E6%97%A0%E9%9C%80%E9%87%8D%E5%90%AF%EF%BC%8C%E8%A7%A3%E5%86%B3%E7%A7%BB%E9%99%A4PCIE%E7%9B%B4%E9%80%9A%E5%90%8E%E6%89%BE%E4%B8%8D%E5%88%B0%E7%A1%AC%E7%9B%98%E7%9A%84%E9%97%AE%E9%A2%98)
