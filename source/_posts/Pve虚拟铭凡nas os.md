---
abbrlink: ''
categories:
- - Pve
date: '2026-03-07T16:33:13.616781+08:00'
tags:
- Pve
title: Pve虚拟铭凡nas os
updated: '2026-03-07T17:15:17.054+08:00'
---
# 第一步，获取序列号

铭凡nas os会检查bios的序列号对不对，不对的话是无法启动的，所以我们需要给虚拟机注入正确的序列号

进入Pve的命令终端

```shell
dmidecode -s system-serial-number
xxxxxxxxxxx
```

这一串就是我们的序列号了

# 第二步，新建一个虚拟机

不要任何的系统，选q35和efi。创建完成后把虚拟机硬件的全部硬盘分离并移除

然后继续回到命令行终端

```shell
nano /etc/pve/qemu-server/xxx.conf
# 虚拟机是xxx号，这里就是xxx.conf
```

```toml
agent: 1
args: -smbios type=1,serial=xxxxxxxxxxx
bios: ovmf
```

在bios: ovmf上面添加一行,xxxxxxx就是第一步获得的序列号

# 第三步，下载铭凡nas os和img2kvm并上传到Pve的root目录

注意，上传铭凡nas os之前，需要把它解压成img格式，再压缩成gz格式上传

因为img2kvm只支持gz,我也不想去改源码，就这么用吧

回到Pve的命令终端

```shell
chmod +x ./img2kvm
```

```shell
./img2kvm nasos-2.1.10.0002.img.gz xxx
xxx是虚拟机号，比如我的是106，那就是./img2kvm nasos-2.1.10.0002.img.gz 106
```

# 第四步，新建一个虚拟硬盘给虚拟机

没有第二块硬盘是无法创建存储池进入系统的，所以需要创建一块硬盘

在硬件--->添加--->硬盘         大小自定义，其他默认

如果显示的是未使用的磁盘，双击然后点添加

# 第五步，开机，用铭凡的app连接虚拟机初始化

**在这一过程，虚拟机可能会自动关机，这是正常，只需要再次启动即可**

## 其他问题

如果创建存储池一直错误，就把原来创建的硬盘分离并移除，然后重新建一块硬盘

[铭凡nas os]([https://www.minisforum.cn/pages/product-info](https://www.minisforum.cn/pages/product-info))

[img2kvm](https://fw.koolcenter.com/binary/other-tools/)
