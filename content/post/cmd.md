---
title: 命令行使用SPD读写工具
description: 工具用到的软件。
date: 2023-1-27 00:00:00+0000
slug: cmd
categories:
    - cmd
tags:
    - cmd
weight: 1
---

# 通过命令行使用读写工具

## 开启终端窗口方法

- `win`+`R`键输入`cmd`回车即可打开终端窗口，`cd D:`进入`D`盘，`cd test`进入`test`文件夹。
- `windows11`可在程序目录右键`在终端中打开`。

## 查看端口

```
./spdrwcli.exe /find
```

返回如下，`COM3`就是需要用到的端口。

```
Found Device on Serial Port: COM3
```

## 查看地址

```
./spdrwcli.exe /scan COM3
```

返回如下，`80`就是需要用到的地址。

```
Found EEPROM at address: 80
```

## 读取SPD

```
./spdrwcli.exe /read COM3 80
```

## 读取SPD并保存为`bin`文件

```
./spdrwcli.exe /read COM3 80 C:\temp\spdoutput.bin
```

## 写入SPD

```
./spdrwcli.exe /write COM3 80 C:\temp\newspd.bin
```

## 启用写保护

```
./spdrwcli.exe /enablewriteprotection COM3 80
```

## 解除写保护

解除端口`COM3`下的所有地址

```
./spdrwcli.exe /disablewriteprotection COM3
```

解除端口`COM3`下的指定地址

```
./spdrwcli.exe /disablewriteprotection COM3 80
```