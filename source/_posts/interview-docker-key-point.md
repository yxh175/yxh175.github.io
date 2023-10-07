---
title: 面试重点：docker常见面试题
date: 2023-10-07 12:48:41
categories: 
- 八股文
tags:
- docker
---

docker平时有用到一点，但并不是经常使用，只用过几个常用的命令，所以还是总结一下常见的docker面试题

## 什么是docker

Go语言开发的一个容器虚拟化技术，使用C/S架构，具有隔离、快速、轻便的特点

## Docker与虚拟机有什么不同？

虚拟机是模拟一个os系统去操作应用，这样的好处是简单。可以很容易的设置各种细节，并且用户也很容易理解多个系统之间的关系。docker不同，它取消了运行臃肿的客户机操作系统，取而代之的是守护进程。通过寄宿在主机中，可以直接与主系统进行通信，并为各个容器分配资源。一个容器就是一个隔绝的沙箱，其中包含着各个镜像。

## Docker镜像是什么？

镜像可以理解为一个可以运行在docker容器中软件包，它包含了运行应用程序所需要的一切代码、运行时环境、系统工具、系统库以及预设的环境变量。镜像可以看作是容器的模板，容器是基于镜像创建的运行实例。

## Docker容器是什么？

Docker 容器是 Docker 平台上运行的独立的、可执行的应用程序单元。容器包含了应用程序及其所有依赖项，包括代码、运行时环境、系统工具、系统库以及预设的环境变量。容器是基于 Docker 镜像创建的实例，它们可以在任何支持 Docker 的环境中运行，而不受环境的影响。

## Docker容器有几种状态？

- 运行
- 暂停
- 重启
- 退出

## DockerFile常见指令

dockerFile常见命令如下

1. FROM: 指定基础镜像，表示从哪里开始构建镜像。

    ```DockerFile
    FROM ubuntu:20.04
    ```

2. MAINTAINER：指定镜像的作者信息。已经不推荐使用，一般可以用 LABEL 代替。
3. LABEL：为镜像添加元数据，通常用于提供关于镜像的信息，如版本、描述、维护者等。例如

    ```DockerFile
    LABEL version="1.0"
    LABEL description="My custom image"
    LABEL maintainer="user@example.com"
    ```

4. RUN：在容器内执行命令，用于安装软件、配置环境等。

    ```DockerFile
    RUN apt-get update && apt-get install -y package-name
    ```

5. COPY 和 ADD：用于将文件从主机复制到容器内。COPY 用于复制文件和目录，而 ADD 可以用于复制文件、目录，并且支持解压缩。

   ```DockerFile
    COPY source_file /destination/
    ADD source_file.tar.gz /destination/
    ```

6. WORKDIR：设置容器内的工作目录，后续的命令都将在这个目录下执行。

   ```DockerFile
    WORKDIR /app
    ```

7. ENV：设置环境变量。

   ```DockerFile
    ENV MY_VAR=value
    ```

8. EXPOSE：声明容器在运行时监听的端口，但并不映射到宿主机上。

   ```DockerFile
    EXPOSE 80
    ```

9. CMD 和 ENTRYPOINT：用于设置容器启动时执行的命令。CMD 通常用于指定容器的默认命令，而 ENTRYPOINT 用于定义容器的入口点，可以与 CMD 结合使用。

    ```DockerFile
    CMD ["app", "-d"]
    ENTRYPOINT ["my-entrypoint-script"]
    ```

10. VOLUME：声明容器内的目录将被挂载为数据卷，用于持久化数据。

    ```DockerFile
    VOLUME /data
    ```

想要了解更多，这个网站有一些基础命令[菜鸟docker教程](https://www.runoob.com/docker/docker-dockerfile.html)

## DockerFile中命令COPY和ADD命令有什么区别？

两者都是可以将文件从主机复制到容器内。但是ADD额外增加解压的功能

## Docker常用命令

常用的Docker命令主要分为五个部分：

- 镜像相关
- 容器相关
- 容器和镜像管理
- 网络相关命令
- Docker Compose

### 镜像相关

```shell
# 从Docker镜像仓库
docker pull IMAGE_NAME:TAG

# 列出本地已经下载的镜像
docker images
docker images ls

# 删除本地镜像 rmi
docker rmi IMAGE_ID

# 根据Dockerfile构建镜像 
# -i: 交互式操作。
# -t: 终端。
docker build -t IMAGE_NAME:TAG PATH_TO_DOCKERFILE
```

### 容器相关命令

```shell

# 创建并运行容器。
docker run -d IMAGE_NAME:TAG

# 列出正在运行的容器。
docker ps

# 列出所有容器，包括已停止的。
docker ps -a

# 启动已停止的容器。
docker start CONTAINER_ID

# 停止运行中的容器
docker stop CONTAINER_ID

# 重启容器。
docker restart CONTAINER_ID

# 在容器内部执行交互式命令。
docker exec -it CONTAINER_ID /bin/bash

# 查看容器的日志输出。
docker logs CONTAINER_ID

# 删除容器。
docker rm CONTAINER_ID

# 清理所有停止的容器。
docker container prune

```

### 容器和镜像管理

感觉有点运维要用的，主要是对容器和镜像的管理

```shell
# 查看容器或镜像的详细信息。
docker inspect CONTAINER_OR_IMAGE_ID

# 给镜像添加标签。
docker tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]

# 基于容器的当前状态创建新镜像。
docker commit CONTAINER_ID NEW_IMAGE_NAME:TAG

# 将镜像保存为 tar 文件
docker save -o OUTPUT_FILE.tar IMAGE_NAME:TAG

# 从 tar 文件加载镜像。
docker load -i INPUT_FILE.tar
```

### 网络相关命令

```shell
# 列出所有 Docker 网络。
docker network ls

# 创建一个自定义网络。
docker network create NETWORK_NAME

# 将容器连接到指定网络。
docker network connect NETWORK_NAME CONTAINER_NAME

# 从网络中断开容器。
docker network disconnect NETWORK_NAME CONTAINER_NAME
```

### Docker Compose

Docker Compose 是一个用于定义和运行多容器 Docker 应用程序的工具，常用命令包括：

```shell
# 启动应用程序。
docker-compose up

# 停止并删除应用程序的容器和网络。
docker-compose down

# 构建 Docker Compose 文件中定义的服务。
docker-compose build
```

当然这些是一些常用的命令，在工作中如果我们需要某些特殊的命令，可以尝试百度或者通过docker --help查找

## 容器与主机之间的数据拷贝命令

docker 主机和容器之间数据拷贝通常也是重要的，比如配置一个nginx时，通过数据拷贝命令，能更方便的把配置修改，而不需要每次启动都去进入容器去修改

### docker cp 方式

docker cp的方法就是直接复制文件到两者的两个方向

- 从容器拷贝到主机
  
    ```shell
    docker cp CONTAINER_ID:/path/to/container/file /path/on/host
    ```

- 从主机拷贝到容器

    ```shell
    docker cp CONTAINER_ID:/path/to/container/file /path/on/host
    ```

这样虽然不用打开容器，但每次修改完后都需要在执行一下复制命令，有没有什么比较好的方法去解决它呢？答案是有的，**共享数据卷**就是一个很好的解决方法

### 共享数据卷

Docker允许通过挂载数据卷的方式在容器和主机之间共享数据，将主机的目录挂载在容器内部，这样就能实现实时共享数据

- 在容器启动时挂载主机目录到容器：

    ```shell
    docker run -v /path/on/host:/path/in/container IMAGE_NAME
    ```

- 在容器内部创建数据卷并挂载到主机：

    ```shell
    docker run -v /path/in/container IMAGE_NAME
    docker cp CONTAINER_ID:/path/in/container /path/on/host
    ```

### 使用共享数据卷容器

还有一种方法，通过创建一个容器作为数据管理中心。创建一个专门的数据卷容器，其他容器可以通过 `--volumes-from `参数来共享数据卷容器的数据卷。这种方式可以用于在多个容器之间共享数据。

- 创建数据卷容器

    ```shell
    docker create -v /data --name data-container IMAGE_NAME
    ```

- 启动其他容器并共享数据卷

    ```shell
    docker run --volumes-from data-container -it ubuntu
    ```

## docker 使用nginx的主要流程

- 首先获取镜像

```shell
docker pull nginx:latest
```

- 查看本地镜像

```shell
docker images
```

- 创建容器运行并挂载路径

```shell
docker run --name nginx-test -p 8080:80 -d -v <主机路径> <容器路径> nginx  
```

## 什么是Docker Swarm

Docker Swarm 是 Docker 的一个集群和编排工具，用于管理和编排多个 Docker 容器的部署。它允许将多个 Docker 主机组成一个集群，并将容器应用程序在这个集群中进行分布式部署和管理。

## 如何批量清理临时镜像文件

```shell
docker rmi -f $(docker images -q)
```

- `-q` 选项表示只输出镜像的 `ID`，而不包括其他信息。
- `$(...)：`这是命令替换的语法，它会将 `docker images -q`的输出结果作为参数传递给下一个命令。
- `docker rmi -f`：这部分命令用于删除` Docker `镜像。`-f`选项表示强制删除，即使镜像正在被容器使用也会被删除。

## 如何查看镜像支持的环境变量？

```shell
# 在 Docker 容器中运行一个镜像，并显示容器中的环境变量。
docker run <镜像ID> env
```

- `docker run`：这是 `Docker` 命令的一部分，用于启动一个新的容器。
- `<镜像ID>`：这是要运行的 `Docker` 镜像的 `ID`。
- `env`：这是在容器内部执行的命令。在这种情况下，它是一个简单的命令 `env`，用于显示容器的环境变量。

## 构建Docker镜像应该遵循哪些原则?

1. 尽量选取满足需求但较小的基础系统镜像
2. 清理编译生成文件、安装包的缓存等临时文件
3. 安装各个软件时候要指定准确的版本号，并避免引入不需要的依赖
4. 从安全的角度考虑，应用尽量使用系统的库和依赖
5. 使用Dockerfile创建镜像时候要添加.dockerignore文件或使用干净的工作目录

## 如何停止所有正在运行的容器?

```shell
docker kill $(docker ps -q)
```

- `docker ps -q`：这部分命令用于列出当前正在运行的 `Docker` 容器的 `ID`，其中 `-q` 选项表示只输出容器的 `ID`，而不包括其他信息。
- `$(...)`：这是命令替换的语法，它会将 `docker ps -q` 的输出结果（即正在运行的容器的 ID 列表）作为参数传递给下一个命令。
- `docker kill`：这部分命令用于停止 `Docker` 容器。当你执行 `docker kill CONTAINER_ID` 时，它会向指定的容器发送一个 SIGKILL 信号，强制停止容器的运行。

整个命令的作用是将 `docker ps -q` 列出的所有正在运行的容器的 `ID` 传递给 `docker kill` 命令，从而停止所有运行中的 `Docker` 容器。

## 如何清理批量后台停止的容器

```shell
# Docker 将删除所有已停止的容器，释放它们占用的资源
docker container prune
```

## 可以在一个容器中同时运行多个应用进程吗？

可以但不推荐

资源不好管理也容易出现冲突，监控也可能会出现冲突

## 如何控制容器占用系统资源（CPU，内存）的份额？

### CPU控制

- `--cpus` 参数：使用 `--cpus` 参数可以限制容器使用的 `CPU` 核心数量。例如，以下命令限制容器只能使用 0.5 个 `CPU` 核心：

    ```shell
    docker run --cpus=0.5 my_container
    ```

- `--cpu-shares` 参数：使用 `--cpu-shares` 参数可以为容器分配 CPU 时间片的权重。较高的权重值表示容器将获得更多的 CPU 时间。例如，以下命令为容器分配了相对较高的 CPU 权重

    ```shell
        docker run --cpu-shares=512 my_container
    ```

### 内存控制

- `--memory` 参数：使用 `--memory` 参数可以限制容器使用的内存量。例如，以下命令限制容器最多使用 512MB 内存：

    ```shell
    docker run --memory=512m my_container
    ```

- `--memory-swap` 参数：使用 `--memory-swap` 参数可以限制容器的虚拟内存（包括物理内存和交换空间）使用。默认情况下，Docker 会将 `--memory-swap` 设置为无限制。例如，以下命令将容器的虚拟内存限制为 1GB（包括 512MB 物理内存和 512MB 交换空间）

    ```shell
    docker run --memory=512m --memory-swap=1g my_container
    ```

## 如何将一台宿主机的docker环境迁移到另外一台宿主机？

停止Docker服务，将整个docker存储文件复制到另外一台宿主机上，然后调整另外一台宿主机的配置即可

## 什么是docker-compose

使用 Docker Compose，你可以在本地开发和测试多容器应用程序，然后将相同的配置和定义部署到生产环境中，从而简化了容器化应用程序的开发、部署和管理过程。它是 Docker 生态系统中非常有用的工具之一。