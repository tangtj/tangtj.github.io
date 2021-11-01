---
layout: post
title: 构建 CICD 遇到的 Docker in Docker 问题
category: 工作
tags:
  - blog
typora-root-url: ..
id: 9
---



最近做的事情。就是使用 gitlab ci/cd ，自建 runner 实现一些自动打包、部署、代码检查等功能。

## 问题

刚开始做就很自然得用 docker 版 gitlab runner 了。ci job 默认使用 docker 镜像，需要在 docker 中进行 docker build，失败了，提示：

```shell
Error during connect: Post http://docker:2375/v1.40/auth: dial tcp: lookup docker on 169.254.169.254:53: no such host
```

这类问题。查阅资料，发现我是遇到了 docker in docker 的问题。

**我需要在 Docker container 里面来构建（build）与运行（run）我们的Docker镜像。**



## 解决方案

### 使用宿主机 docker


Docker 是 CS 模式的，在Docker Client中来运行Docker的各种命令，而这些命令会传送给在Docker的宿主机上运行的Docker的守护进程。而Docker的守护进程是负责来实现Docker的各种功能。




![image-9-1.png](/file/image/9-1.png)



默认情况下，Docker 守护进程会生成一个 socket（/var/run/docker.sock）文件来进行本地进程通信，而不会监听任何端口，因此只能在本地使用docker客户端或者使用Docker API进行操作。一般情况下，我们访问本机的服务往往通过 **127.0.0.1:8080** 这种 IP:PORT 的网络地址方式进行通信，而sock文件是 **UNIX 域套接字**（UNIX domain socket），它可以通过文件系统（而非网络地址）进行寻址和访问的套接字。

从表象上看，上面的命令似乎依然是在“Docker里面run docker”，其实这是个误区。docker run提供了 **-v** 参数让我们将宿主的文件映射到docker里面。比如通过 **-v /var/run/docker.sock:/var/run/docker.sock**，我们将宿主的Docker Daemon的socket映射到Docker Container里面；当Container里面的docker 客户端通过 /var/run/docker.sock 去操作Docker Daemon时，这些操作已移花接木地转移到宿主的Docker Daemon上。



具体如下：

```shell
docker run --name docker -v /var/run/docker.sock:/var/run/docker.sock --privileged -d docker
```

启动特权模式，并且挂载 **/var/run/docker.sock** 到容器内。即可在容器中使用宿主机的Docker Daemon。

可以通过：

```shell
docker pull redis
```

命令测试，随机拉取一个镜像，然后在宿主机进行：

```shell
docker image ls
```

 查看是不是存在刚来拉下来的镜像进行验证。



提示：如果你像我一样，用 docker 版本的 runner 起了一个 docker 容器，然后在 docker 容器中进行 docker build ，包了四五层的话。你需要将  **/var/run/docker.sock**  从外到内一直进行传递，直到最内层。



**需要注意的点**

因为是直接操作的宿主机 docker，所以容器内的操作会直接影响到宿主机，需注意安全，谨慎操作

- 使用的宿主机 docker 所以 build 的时候是能使用到 cache
- 在内层 -v 挂载时的目录，实际上是在挂载宿主机目录。eg  **-v /data/app/data1:/data/app/data2** ，在 docker 中 有构建出二进制，想挂载到 下一层docker 中当作资源时，因为是用的宿主机 docker daemon，所以这个 /data/app/data1 的路径是宿主机的路径，并不是 docker 容器中的路径
- 在 docker 容器中进行的 docker run/stop/restart 命令，会真实影响到宿主机执行



### 使用 docker:dind 镜像



如果场景是想隔离资源，并在容器内使用docker命令的话，可以使用 docker:dind。

docker:dind 该镜像包含 Docker 客户端（命令行工具）和 Docker daemon。

通过 docker history docker:dind 命令我们发现 docker:dind 是在 docker:latest 基础上又安装了 Docker daemon

启动 `docker:dind` 容器时，参数 `--privileged` 必须加上，否则 Docker daemon 启动时会报错。

```shell
docker run -d --name dind --privileged docker:dind # 启动容器
```

同样的通过尝试拉取 image 在宿主机的 docker 进行验证是否隔离。

这种方式，docker 容器内是与宿主机隔离的。所以不会影响宿主机容器的状态，在构建时也无法应用上缓存。
