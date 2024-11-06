---
title: Windows下使用NVM管理Node.js版本
tags:
  - Node.js
date: 2024-11-06 20:28:10
---


# 一、背景

不同项目可能需要不同版本的Node.js或者想要体验不同版本的Node.js，这时候如果手动下载各个版本Node.js再配置环境变量就非常繁琐。这种情况下可以使用一些Node.js的版本管理工具进行管理，替我们去下载并配置Node.js，本篇文章使用NVM进行Node.js多版本管理。

# 二、下载NVM

注意：安装nvm前推荐卸载先前安装的Node.js，可以避免一些问题。

nvm官网：https://github.com/nvm-sh/nvm

nvm-windows官网：https://github.com/coreybutler/nvm-windows

nvm-windows下载地址：https://github.com/coreybutler/nvm-windows/releases

这里提供百度网盘的下载链接：https://pan.baidu.com/s/1AQIhv4bL1KXe_Hm6vVB8KA?pwd=qg9v


这里下载非安装版本手动安装，下载后再配置环境变量。

{% asset_img "nvm-windows下载.png" "nvm-windows download" %}

# 三、手动安装NVM

## 1、解压文件

解压文件到要安装的目录，`nvm.exe`、`elevate.vbs` 以及 `elevate.cmd` 都是 NVM 正常运行必需的。"elevate"脚本提升操作所需要的权限，这是切换 Node.js 版本时需要的重要组件。

> Windows 下的 NVM 通过使用 mklink 命令来更新 symlink 来切换 Node.js 版本。重新创建的符号链接指向运行的 Node.js 版本，这个操作需要较高的管理权限，所以 "elevate" 脚本至关重要。

{% asset_img "nvm-windows文件.png" "nvm-windows fies" %}

## 2、配置环境变量

需要创建 `NVM_HOME`、`NVM_SYMLINK` 两个环境变量，`NVM_HOME` 指向 NVM 的安装路径，`NVM_SYMLINK` 指向当前正在运行的哪个版本的 Node.js 路径，这是一个目录链接，**我们只需要配置环境变量，不需要创建文件夹，应该有 NVM 自动创建和维护**。这里以E盘为例子。

{% asset_img "E盘文件夹.png" "E盘中的文件夹" %}

{% asset_img "环境变量.png" "nvm-windows environment variable" %}

**注意：nodejs 文件夹不用创建，应该由 NVM 自动创建和维护**

为了更方便的执行命令，我们将运行文件的路径添加到 `Path` 中

{% asset_img "Path中的内容.png" "Path中的内容" %}

## 3、配置 NVM

在 NVM 的安装目录下创建 settings.txt 配置文件。

该文件中的关键属性：

- root：安装目录(`NVM_HOME`)

- path：符号链接目录 `NVM_SYMLINK`

- proxy：代理，设置成"none"。如果需要代理，可以使用 nvm 命令修改

- arch：`32` 或 `64`，取决于操作系统是 32 位还是 64 位

文件内容如下：

```
root: E:\nvm
path: E:\nodejs
arch: 64
proxy: none
```

好了，接下来可以使用 NVM 对 Node.js 进行版本管理了。


# 四、使用 NVM

打开 CMD 窗口，输入 nvm 即可看到相关信息。

{% asset_img "nvm命令.png" "nvm-windows fies" %}

一些常用命令如下：

|命令|说明|
|-----|---|
|nvm current|当前使用的Node.js版本|
|nvm install|按照指定版本的Node.js|
|nvm list|罗列已经安装的Node.js版本|
|nvm list available|罗列可以安装的版本|
|nvm use|使用指定版本|

1. 列出可以按照的版本：

{% asset_img "列出可以安装的版本.png" "列出可以安装的版本" %}

2. 安装指定的版本

以安装 18.20.4 为例：

{% asset_img "NVM安装Node.js命令.png" "NVM安装Node.js命令" %}

安装Node.js后 NVM 的安装文件夹中出现了 v18.20.4 文件夹，其实就是我们下载的 Node.js。

{% asset_img "安装Node.js后文件中的内容.png" "安装Node.js后文件中的内容" %}

指定使用Node.js的版本`nvm use 18.20.4`，会发现出现了我们先前配置的 `NVM_SYMLINK`文件夹：

{% asset_img "NVM_SYMLINK文件夹.png" "NVM_SYMLINK文件夹" %}

查看文件属性：

{% asset_img "NVM_SYMLINK文件夹属性.png" "NVM_SYMLINK文件夹属性" %}

可以发现这个文件夹是一个符号链接，指向我们下载的Node.js

# 五、卸载

可以按照以下步骤手动卸载（无特定顺序）：

1. 删除根安装目录（`NVM_HOME`）。这会删除所有版本的 Node、npm 和 Windows 核心 NVM 文件。

2. 删除符号链接（`NVM_SYMLINK`）。只有我们安装了至少一个 Node 版本且使用 nvm use 指定了版本才会存在。

3. 删除环境变量（`NVM_HOME`和`NVM_SYMLINK`）并清理 `Path` 中的内容。
