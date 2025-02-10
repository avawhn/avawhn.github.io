---
title: windows上kafka报错:log4j:ERROR Failed to rename xx to xx
tags:
  - Kafka
date: 2025-02-10 22:09:18
---


这个问题在网上已有文章，这里仅作为记录。

## 问题描述

windows 上部署好 kafka，运行一段时间后报错 `log4j:ERROR Failed to rename [C:\tools\kafka/logs/controller.log] to [C:\tools\kafka/logs/controller.log.2024-12-02-14].`

{% asset_img "kafka错误日志.png" "kafka错误日志" %}


## 原因

[Kafka错误：window下运行一段时间后自动挂掉/宕机](https://www.cnblogs.com/bayu/articles/14467236.html)，这篇文章指出是因为过期策略触发重命名，这与报错信息中的两个文件名格式相对应。


## 解决方案

关于这个问题，目前官方没有解决方案，Github 上 这个[PR](https://github.com/apache/kafka/pull/6329)里面有人提交了补丁，官方认为不稳定，存在问题，并没有合并。

解决方案：

1. 过期策略永不过期
2. 改用 Linux 系统（WSL、虚拟机）
3. 使用补丁
4. 修改 log4j 源码，将这部分重命名改为复制到新文件并清空内容：https://blog.csdn.net/qq_41600330/article/details/123525153?ydreferer=aHR0cHM6Ly93d3cuZ29vZ2xlLmNvbS8%3D

对于上面的解决方案，项目目前情况是，

1. 消息产生较多，磁盘占用增加过快（不合适）
2. 项目是部署在用户（厂家）服务器上，硬件目前充足，WSL占用内存影响不大（可以选择）
3. 项目对于 Kafka 的使用场景比较简单，采用 PR 中的补丁问题不大，[Kafka错误：window下运行一段时间后自动挂掉/宕机](https://www.cnblogs.com/bayu/articles/14467236.html) 这篇文章的作者也提供了打包好的文件（可以选择）
4. 修改 log4j 源码的解决方案，有点粗糙（不合适）

综合考虑，使用 WSL 来部署 Kafka，后续硬件吃紧在考虑使用打补丁的文件


## 参考文章

https://github.com/apache/kafka/pull/6329

https://www.cnblogs.com/bayu/articles/14467236.html

https://blog.csdn.net/qq_41600330/article/details/123525153?ydreferer=aHR0cHM6Ly93d3cuZ29vZ2xlLmNvbS8%3D

 