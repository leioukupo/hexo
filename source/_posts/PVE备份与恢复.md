---
abbrlink: ''
categories:
- - pve
date: ''
tags:
- pve
title: PVE
updated: '2025-03-31T10:26:33.031+08:00'
---
# PVE的备份和恢复

进入pve管理后台，在各虚拟机的备份选项点击备份

备份文件路径是*Var/lib/vz/dump*，下载到其他地方保持

需要恢复时将备份文件上传到*Var/lib/vz/dump*，在存储 local里点击对应的备份文件进行还原，然后启动虚拟机

！！！如果启动虚拟机时提示找不到iso镜像，找到硬件选项，编辑CD/DVD驱动器重新选择一下iso镜像文件

## pve直通amd核显没法玩

直接装fedora,然后用fedora的虚拟机装pve
