---
title: Docker部署SpringBoot项目
date: 2024-01-02 15:53:14
tags: [SpringBoot,Docker]
---

创建 spring-boot-demo 文件夹，复制 jar 包到该文件夹下面，并创建 dockerfile 文件

{% asset_img 文件.png 文件.png %}

dockerfile 如下：

```dockerfile
# 基础镜像
FROM openjdk:8-jdk-alpine

# 作者
LABEL author="whn"

# 将 jar 包添加到容器中
ADD demo-1.0-SNAPSHOT.jar /root

# 暴露 8080 端口
EXPOSE 8080

# 默认命令
CMD java -jar /root/demo-1.0-SNAPSHOT.jar
```

构建镜像：

```sh
docker build . -t demo:v1
```

{% asset_img 构建镜像.png 构建镜像.png %}

运行实例：

```sh
docker run --name demo -p 8080:8080 -d demo:v1
```

查看是否启动成功：

```sh
docker ps
```

查看日志：

```sh
docker logs demo
```

{% asset_img 查看日志.png 查看日志.png %}


测试：

{% asset_img 测试.png 测试.png %}


