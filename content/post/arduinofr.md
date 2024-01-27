---
title: Arduino烧录固件
description: 包含提取固件，写入固件，IDE安装环境及Arduino USB驱动。
date: 2024-1-27 00:00:00+0000
slug: fr
categories:
    - firmware
tags:
    - firmware
weight: 1
---

# Arduino烧录固件

## 提取固件

打开SPDRW文件夹 双击 `spdrwgui` 

![spdrwgui-1](post/img/spdrwgui-1.png)


依次点击如下图：

![spdrwgui-1](post/img/spdrwgui-2.png)
![spdrwgui-1](post/img/spdrwgui-3.png)
![spdrwgui-1](post/img/spdrwgui-4.png)


## 烧录固件

PC与Arduino连接，打开任务管理器，查看端口，然后等待5分钟左右会自动安装驱动，出现CH340类似字样说明安装成功，打开Arduino IDE，打开后注意IDE下方显示框所显示的环境安装，等待它安装完成，大概5分钟左右，然后如下图添加Arduino。

![spdrwgui-1](post/img/arduino-1.png)

打开固件，选择刚才提取的固件ino文件

![spdrwgui-1](post/img/arduino-2.png)
![spdrwgui-1](post/img/arduino-3.png)
![spdrwgui-1](post/img/arduino-4.png)

确认文件无误后点击“右指向箭头图标”上传。

![spdrwgui-1](post/img/arduino-5.png)

上传成功如下图：

![spdrwgui-1](post/img/arduino-6.png)

至此固件上传完毕
