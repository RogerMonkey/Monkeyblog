---
title: 双系统mbr分区grub引导启动问题解决指北
date: 2017-03-25 20:15:46
tags: 
  - Linux
  - grub
categories: 
  - Linux
  - 其他
---
## 问题
相信有很多人和我一样，喜欢用linux敲代码和工作，却抵挡不了windows的诱（you）惑（xi）而选择了双系统（买mac的壕们请无视）。  
那么问题就来了，很多时候由于安装不当，启动时只能找到一个系统，所以在此列举一些应对措施。  

<!-- more -->
## 安装linux启动不了windows
大部分linux启动文件都使用grub或grub2。如果安装双系统后启动电脑启动不了windows，可以进入linux修复引导。  
进入linux，使用如下命令即可更新grub自动找回启动项：`sudo update-grub2`，之后重启电脑即可。

## 双系统误操作黑屏找不到分区
### PE启动
- 下载老毛桃或者大白菜
- u盘启动（F12 或其他）进入dg分区管理
- 修复主分区mbr，即可重新引导
- 咳咳，一般没用

### linux liveCD启动
- 下载liveCD linux从u盘启动进入系统
- `sudo -i`以下命令都要在root权限下执行
- `fdisk -l`查看硬盘分区，找到/和/boot分区号（如/dev/sda1 /dev/sda2)
- `mount /dev/sda1 /mnt` `mount /dev/sda2/ /mnt/boot`挂载这两个分区
- 在硬盘重建mbr分区表，执行以下命令 `gurb2-install --root-directory=/dev/sda` PS.这里路径是硬盘号，不是分区号
- 重启～

### grub rescue
win+linux双系统下若进行重装linux，修改硬盘分区等操作时可能会出现启动电脑黑屏，提示无法找到分区，此时会进入类似shell的grub rescue界面。  
这个模式命令只有`ls`,`set`,`insmod`,`root`,`prefix`,`normal`等可用。  
如果linux系统还在，那么还有救，步骤如下:
- `ls`查看硬盘信息，如(h0,msdos1)既为一个分区
- `ls (hd0,msdos1)/boot/grub/`或`ls (hd0,msdos1)/grub/`找寻存放grub的boot分区
- `set root=(hd0,msdos1)`设置该分区为root
- `set prefix=(hd0,msdos1)/boot/grub`设置启动项
- `insmode normal`进入grub菜单，如果成功，即可进入linux系统
- 照第一节内容更新grub，即可解决问题

## 总结
这里只说到了MBR分区下的启动问题，当然GPT分区启动也可能会出现问题，总之安装双系统坑很多。  
以后也许可以抽空写一下分区表和启动的区别，当然得有空。希望实验顺利～
