---
layout:     post
title:      "2019-04-16-DockerFile命令"
subtitle:   "docker学习记录"
date:       2019-04-16
author:     "CHuiL"
header-img: "img/k8s-bg.png"
tags:
    - docker
---

### DockerFile
用来快速创建自定义镜像

一般DockerFile分为四部分：基础镜像信息，维护者信息，镜像操作指令和容器启动时执行指令。


#### FROM命令
==任何DockerFile第一行都必须是FROM命令==，表示指定所创建的基础镜像，如果本地没有会从Docker Hub上拉取；多个可以多条；

#### MATINTAINER命令

 `MAINTAINER image_creator@docker.com`
 该命令会将该信息写入镜像的Author属性域中
 
#### RUN命令
运行指定命令；格式为  
`RUN<command>`或`RUN<"executable","param1","param2">`;前者默认将在shell终端中运行命令，即/bin/sh -C；后者使用exec执行，不会启动shell环境；  
　==每条RUN指令将在当前镜像的基础上执行指定命令，并提交为新的镜像，每RUN一条，镜像就添加新的一层==。当命令较长时可以使用\来换行；例如
　`RUN apt-get update \ && apt-get install.....\...`
　
#### CMD命令
指定启动时容器默认执行的命令。支持三种格式
- `CMD ["executable","param1","param2"]`
- `CMD command param1 param2`在/bin/sh中执行
- `CMD ["param1","param2"]`提供给ENTRYPOINT的默认参数
每个DockerFiile中只能有一条CMD命令，==多条默认执行最后一条==；如果用户启动容器时手动执行了运行的名（作为run的参数），则会覆盖掉CMD指定的命令

#### LABEL
用来指定生成镜像的==元数据标签信息==
`LABEL <key>=<value> <key>=<value> <key>=<value>`  
例如 `LABEL version="1.0"`; `LABLE description="This text illustrates \ that label-values can span multiple lines."`

#### EXPOSE
声明==镜像内服务所监听的端口==
格式为`EXPOSE <port> [<port>....]`
例如 `EXPOSE 22 80 8443`

#### ENV
指定环境变量；==被后续RUN使用==；镜像启动的容器中也会存在  
格式为`ENV <key> <value>`或 `ENV<key>=<value>...`  
例如`ENV PG_MAJOR 9.3`;`ENV PG_VERSION 9.3.4` ;`ENV PATH /usr/local/postgres-$PG_MAJOR/bin:$PATH`
#### ADD
复制指定的<src>路径下的内容到容器中的<dest>路径下  
格式为：`ADD <src> <dest>`，其中<src>可以是Dockerfile所在目录的一个相对路径，也可以是一个URL,也可以是一个tar文件（==如果为tar文件，将自动解压==），dest可以是镜像内绝对路径，也可以是相对工作目录（WORKDIR）的相对路径；  
例如：`ADD *.c/ /code/`

#### COPY 
 格式为`COPY <src> <dest>`；复制本地主机的<src>（为dockerfile所在目录的相对路径、文件或目录）下的内容到镜像中dest下。目标路径不存在会自动创建
 ==使用本地目录为源目录时，推荐使用==
 
#### ENTRYPOINT
指定默认入口命令；启动容器时作为根命令执行；所有传入值作为该命令的参数  
格式`ENTRYPOINT ["executable","param1","param2"]`(exec调用执行)
 `ENTRYPOINT command param1 param2`;  
 默认执行最后一个ENTRYPOINT;运行时可被--entrypoint覆盖；
 
 #### VOLUME
 创建一个数据挂载点
 格式为`VOLUME ["/data"]`
 可以从本地主机或其他容器挂载数据卷，一般用来存放数据库和需要保存的数据等
 
#### USER
 指定运行容器时的用户名或UID,后续的RUN等指令也会使用指定的用户身份；  
 格式为`USER daemon`
 当服务不需要管理员权限时，可以通过该命令指定运行用户，并且可以在之前创建所需要的用户；例如  
 `RUN groupadd -r postgres && useradd -r -g postrgres postgres`
 
#### WORKDIR
 为后续的RUN、CMD和ENTRYPOINT指令配置工作目录
 格式为`WORKDIR /path/to/workdir`  
 可以使用多个；后续如果使用相对命令，会基于之前命令和指定路径；
 
 #### ARG
 指定一些镜像内使用的参数，这些参数在执行docker build命令时才会以 `--build-arg<varname>=<value>`格式传入  
 
#### ONBUILD
  配置当所创建的镜像作为其他镜像的基础镜像时，所执行的创建操作指令；  
  例如创建了一个镜像A  
  `ONBUILD ADD . /app/src`  
  `ONBUILD RUN /user/.....`  
 这样在另外一个镜像中  
 `FROM　image-A`,会在自动执行上述的两条命令。
 
#### STOPSIGNAL
 指定所创建的镜像启动的容器接受退出的信号值。例如：`STPOSIGNAL signal`；
 
#### HEALTHCHECK
 配置所启动的容器如何检查健康；格式有两种
 `HEALTHCHECK [OPTIONS] CMD command`:根据所执行的命令返回值是否为0来判断  
 `HEALTHCHECK NONE`:禁止基础镜像中的健康检查
 
###### OPTION支持
 - --interval=DURATION(默认30秒)，过多久检查一次
 - --timeout=DURATION(默认30秒),每次检查等待结果的时间
 - --retries=N(默认3),如果失败了，重试几次才最终确定失败

#### SHELL
指定其他命令使用shell时的默认shell类型  
`SHELL ["executable","parameters"]`

### 创建镜像
编写好dockerfile之后，可以使用docker file命令来创建镜像  
格式 `docker build [dockerfile path]`  
执行该命令后将读取路径下的所有内容并发送给docker服务器，由服务端来创建镜像。