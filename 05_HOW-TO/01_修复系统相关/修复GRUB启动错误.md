---
title: 修复GRUB启动错误
description: 
published: true
date: 2023-02-22T09:00:05.009Z
tags: 
editor: markdown
dateCreated: 2022-04-21T03:46:19.671Z
---

## 介绍

如果GRUB接管MBR，那么GRUB安装分为三部分：

* 第一部分（一般情况下）写在了MBR上
* 第二部分是将core.img嵌入到MBR之后的保留扇区部分
* 第三部分才是/boot/grub下面的各种模块和文件（如果/boot单独分区，则直接写在对应分区的/grub目录）里面。

>注意:本条目只针对GRUB2.0。

## 显示GRUB

一般情况下，GRUB菜单是直接显示的。

有时候用户会将GRUB等待时间设为0，若想临时显示GRUB菜单，开机时，在GRUB加载前按住Shift键不放即可，部分主板可能需要多重启一次才会生效。

若能进入深度操作系统，也可以到 控制中心->启动菜单 调整相应选项。

## GRUB错误

### deepin15.3启动失败（UEFI）

#### 报错与分析

```
GRUB loading
Minimal BASH-like line editing is supported.For the first word...(后面省略)
grub > 

```

与旧版教程中的报错不同，这种情况下标识符是grub>而不是grub rescure>。这个时候直接输入normal并回车不会执行任何操作，说明是normal.mod出错导致的，grub正常识别了deepin的/boot分区，但是加载了出错的normal.mod，导致无法引导系统。出错的原因可能是由于easybcd与grub之间存在兼容性问题导致的，也可能是因为之前重复安装其他操作系统系统但是删除旧系统后没有清理efi分区甚至是直接在旧系统上安装deepin15.3。

#### 解决办法

使用liveUSB、liveCD或者设备上的另一个linux发行版打开gparted查看引导出错的deepin15.3的根目录挂载点，例如/dev/sda1，具体的值与你的分区有关，为了描述方便下面均以/dev/sda1为例，实际操作时记得改为你的系统的挂载点，下文的系统分区也是，以你的实际显示为准;
选择deepin引导，进入grub命令行后（也就是这个报错界面），输入set然后回车；
一般这种报错并不是因为路径识别出错，如果你希望验证是否是grub没有正确识别系统路径，可以参照[链接](http://listenerri.com/2016/04/23/uefi-gpt-linux%E4%BF%AE%E5%A4%8Dgrub-rescue/)的“查找正确分区”部分（后面的不需要看，因为错误不同），所以可以直接查看prefix=行的显示（一般在set输出的末尾几行），示例：

    prefix=(hd2,gpt1)/boot/grub

其中(hd2,gpt1)代表系统所在分区。为了描述方便下面均以(hd2,gpt1)为例。
输入 `linux (hd2,gpt1)/boot/vmlinuz`，然后按tab按键（Q键左边的那个按键）补全名字。补全之后不要按回车，而是输入**空格**，然后继续输入`root=/dev/sda1 foo bar`，之后才回车。这一步是加载系统内核。

> 注意：(hd2,gpt1)和/boot之间没有空格。

输入`initrd (hd2,gpt1)/boot/init`，然后按`tab`按键补全名字，补全之后回车。
输入`boot`，回车，就可以引导进入系统了。
进入系统之后，在终端下输入`sudo update-grub`，之后在控制中心打开启动菜单选项，等待它自动更新，更新完毕后就完成修复工作了，可以正常引导deepin15.3了。欢呼吧！

鸣谢：@mattd

### 旧版方法

```
GRUB loading
error:unknow filesystem
grub rescue> 

```

已经发现下面几种操作会导致这种问题：

* 想删除Linux，于是直接在windows下删除/格式化了Linux所在的分区。
* 调整磁盘，利用工具合并/分割/调整/删除分区，使磁盘分区数目发生了变化。
* 重新安装系统，把linux安装到了新分区，原有分区已经格式化，但是没有重新安装GRUB2。
* 用Linux备份工具/衍生版制造工具等，把主分区恢复成了8.X的老版本，结果老版本的GRUB是GRUB1，于是把GRUB2破坏掉了。

### 重装GRUB

1. 使用深度操作系统启动盘引导电脑启动，待进入安装界面后，按下Ctrl+Alt+F1，执行以下命令，稍等片刻，进入Live CD模式。

   ```
   sudo service lightdm stop  
   startx
  
   ```

2. 进入Live CD系统后打开终端，挂载需要修复系统的 / 挂载到/mnt，可以利用Gparted或者 `sudo fdisk -l` 命令查看，例如需要修复系统的/分区为/dev/sda1。

   执行以下命令：

   ```
   sudo mount /dev/sda1 /mnt
   
   ```

   如果需要修复系统的/boot单独分了出来（假设为/dev/sda2），也要挂上，终端执行：

   ```
   sudo mount /dev/sda2 /mnt/boot

   ```

   另外,将Live CD系统的/dev目录同时挂在/mnt下，终端执行：

   ```
   sudo mount --bind /dev /mnt/dev

   ```

   然后使用chroot命令，将Live CD的 / 设为以前的/，终端执行：

   ```
   sudo mount --bind /proc /mnt/proc
   sudo mount --bind /sys /mnt/sys
   sudo chroot /mnt

   ```

   安装并刷新GRUB设置(主板为BIOS引导)，请终端执行：

   ```
   grub-probe -t device /boot/grub
   sudo grub-install /dev/sda
   sudo grub-install --recheck /dev/sda
   sudo update-grub
   ```

   安装并刷新GRUB设置(主板为UEFI引导)：
启动root shell后，检查您的EFI系统分区（最可能 `/dev/sda1`）是否安装在 `/boot /efi` 上：

   ```
   # mount /dev/sda1 /boot/efi
   ```

重新安装grub-efi包：

   ```
   # apt-get install --reinstall grub-efi

   ```

将debian引导加载程序放在 `/boot/efi` 中，并在计算机NVRAM中创建一个适当的条目：

   ```
   # grub-install /dev/sda

   ```

重新创建一个基于磁盘分区模式的grub配置文件：

   ```
   # update-grub
   ```

3. 挂载EFI分区到 `/boot/efi`，安装 `grub-efi` 包：

   ```
   # grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=Deepin
   # grub-mkconfig -o /boot/grub/grub.cfg

   ```

4. 修复完成，重启电脑生效。

### 删除GRUB

删除GRUB可能会导致电脑无法引导deepin，请谨慎操作。如果需要彻底删除GRUB2（卸载深度操作系统），请查看**卸载系统**。

## EFI+GPT模式下修复GRUB2双系统引导
本节内容为转载，[原地址](http://www.mintos.org/skill/rescue-efi-grub.html)

不是所有人都能够只用 Linux 单系统！

目前多数电脑自带的正版 Windows 8/10 都是 EFI 引导 + GPT 分区模式，那么 Windows + Linux 的双系统局面仍将长期存在，Linux 用户再不乐意也还是要适应。最近薄荷站长把常用电脑转换成 EFI 引导 + GPT 分区模式了，现将一些必要的知识分享出来，希望新手朋友少走弯路。

对于双系统用户，一般而言，推荐先安装 Windows 8/10，再安装 Linux，并使用 Linux 的 GRUB2 作为双系统引导管理器。那么，重装 Windows 后，GRUB2 会被破坏，只能进入 Windows。如何再次找回 GRUB2 双系统引导，就是本文的主题。

1. 用 Linux 启动盘进入 Live 系统环境，在 Live 的终端里，创建修复 GRUB2 所需的文件夹：

   ```
   sudo mkdir -p /mnt/system

   ```

2. 把 Linux 的 / 分区挂载到创建的文件夹：（注意：站长的是 sdb4，请确认自己的 / 分区所在，不可照搬）

   ```
   sudo mount /dev/sdb4 /mnt/system

   ```

3. 把 EFI 分区（即 ESP 分区）也挂载：

   ```
   sudo mount /dev/sdb1 /mnt/system/boot/efi

   ```

4. 用 efibootmgr 创建 ubuntu 的启动项：（注意：站长的主硬盘是 sdb，请确认自己的主硬盘，不可照搬）

   ```
   sudo efibootmgr -c -d /dev/sdb -p 2 -w -L ubuntu

   ```

5. 重启，并在 BIOS 中选择刚才创建的 ubuntu 启动项，进入 Ubuntu。

6. OK，已经进入本机硬盘上的 Ubuntu 系统了，但 GRUB2 修复并未完毕。打开终端，重新安装 GRUB2 到 EFI 分区：

   ```
   sudo grub-install /dev/sda1

   ```

7. 刷新一下 GRUB2 配置：

   ```
   sudo update-grub2

   ```

8. 现在重启，即可看到亲切的 GRUB2 终于“夺回”双系统引导权了！

---------------------------------------------
修订：
站长另外介绍一种更简便的方法。用 Linux 启动盘进入 Live 系统环境，在终端中依次执行如下命令：

```
$ sudo su
# mount /dev/sda4 /mnt（注意先确认自己的 / 分区是 sdaX）
# mount /dev/sda1 /mnt/boot/efi
# mount -t proc proc /mnt/proc
# mount -t sysfs sys /mnt/sys
# mount -o bind /dev /mnt/dev
# mount -t devpts pts /mnt/dev/pts/
# chroot /mnt
# grub-install /dev/sda1
# update-grub2
```

小结：EFI 引导 + GPT 分区模式下的双系统问题稍微复杂一点，需要朋友们多实操、多领会，关键是搞清楚自己的硬盘分区（EFI 分区和 / 分区）的作用、在不同系统环境下的名称，切记切记！

## 常见问题
### 开关机动画没有正常显示

1. 修改 `/boot/grub/grub.cfg`。
2. 搜索当前核心，例如：`vmlinuz-4.9.0-deepin4-amd64`。
3. 最后面 `ro  quiet` 修改为 `ro splash quiet`。
4. 重启即可。

## 参考链接

* [如何修复GRUB2](http://linux.cn/home-space-uid-6515-do-blog-id-967.html)
* [ubuntu中文论坛:GRUB Rescue修复方法](http://forum.ubuntu.org.cn/viewtopic.php?f=139&t=348503)
* [最新修复双系统GRUB的方法](http://zhuyalin.cn/2012/08/%E6%9C%80%E6%96%B0%E4%BF%AE%E5%A4%8D%E5%8F%8C%E7%B3%BB%E7%BB%9Fgrub%E7%9A%84%E6%96%B9%E6%B3%95/)
* [Windows硬盘安装deepin及引导修复，安全删除](http://bbs.deepin.org/forum.php?mod=viewthread&tid=11773)
* [如何在Ubuntu12.04/12.10中重装或修复GRUB2引导](http://www.linuxidc.com/Linux/2012-11/74901.htm)
* [deepin15.3开关机Logo不见了，怎么搞出来](https://bbs.deepin.org/forum.php?mod=viewthread&tid=136746#lastpost)
