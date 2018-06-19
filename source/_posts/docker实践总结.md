---
title: docker实践总结
categories: 技术
tags: [docker]
date: 2017/12/30 20:00:00
---

## start

&emsp;&emsp;docker作为日常的部署工具已经用了一年多了,可以说对于开发非常实用.想要部署任何应用,都可以去[docker hub](https://hub.docker.com/)上找到对应的镜像,省去了很多安装的过程.今天就来记录一下自己的使用心得.

- - -

<!--more-->

- - -

## content

### 安装docker

&emsp;&emsp;很多人喜欢在服务器上直接`yum install docker`(CentOS系统),我推荐用脚本安装[https://get.docker.com/](https://get.docker.com/).
```bash
curl -fsSL https://get.docker.com/ | sh
systemctl start docker.service
systemctl enable docker.service
```

### 常用命令

- `docker search [image_name]`:查询镜像,这里查询到的镜像为官方镜像仓库的镜像.
- `docker pull [image_name]:[image_tag]`:拉取镜像,这里注意每个镜像都有自己的版本号,比如一个`tomcat`镜像可能分为`v7`、`v8`等版本.如果直接`docker  pull tomcat`那么下载就是`tomcat:latest`,如果想要下自己想要的版本,可以去`docker hub`上查找镜像的版本号,然后再拉取.
- `docker images`:查看镜像
- `docker ps`:查看已经运行状态的容器,`docker ps -a`可以查看所有容器,包括停止状态的容器.
- `docker rmi [image_name]`:删除镜像,如果存在该镜像启动的容器则可以加上`-f`参数强制删除.
- `docker rm [container_name]`:删除容器,如果容器处于运行状态则可以加上`-f`参数强制删除.
- `docker tag [old_image_name]:[old_image_tag] [new_image_name]:[new_image_tag]`:给镜像重新打标签,虽然是重命名,但是不会创建新的镜像,而是创建新的引用指向原来的镜像.
- `docker stop [container_name]`:停止容器.
- `docker start [container_name]`:启动容器.
- `docker restart [container_name]`:重启容器.
- `docker logs -f --tail [log_size] [container_id]`:查看容器日志
- `docker inspect [container_id]`:查看容器启动属性
- `docker stats`:查看容器资源使用监控情况,类似于linux下的`top`命令.

&emsp;&emsp;常用的命令大概就这些,具体命令参数可以`man docker [command]`查看详情.

### 制作镜像

&emsp;&emsp;这里不详细解释Docker容器、镜像这些概念,不了解的话可以看书、博客,这方面的知识可能需要深入
了解linux内核.现在,我们需要掌握通过Dockerfile构建出Docker镜像,然后再通过Docker镜像启动Docker容器.

- 制作基础镜像:由于`alpine`操作系统镜像体积较小,所以选择它作为基础镜像.

**Dockerfile**
```bash
FROM alpine:3.4
MAINTAINER Jeffrey <ji459705636@163.com>

ENV LC_ALL en_US.UTF-8
ENV LANG en_US.UTF-8
ENV LANGUAGE en_US:en

RUN echo "http://mirrors.aliyun.com/alpine/v3.4/main" > /etc/apk/repositories && \
    echo "http://mirrors.aliyun.com/alpine/v3.4/community" >> /etc/apk/repositories && \
    apk add --no-cache wget && \
    wget -O /usr/local/bin/dumb-init https://github.com/Yelp/dumb-init/releases/download/v1.2.1/dumb-init_1.2.1_amd64 && \
    chmod +x /usr/bin/dumb-init && \
    apk add --no-cache tzdata && \
    ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
    echo "Asia/Shanghai" > /etc/timezone
```

```bash
docker build -t baseimage-alpine:v3.4 .
```

&emsp;&emsp;在`alpine`操作系统的基础上安装了`dumb-init`和`tzdata`,`tzdata`用于作为时间同步.容器化环境中,往往直接运行应用程序,而缺少初始化系统(如systemd、sysvinit等).这可能需要应用程序来处理系统信号,接管子进程,进而导致容器无法停止、产生僵尸进程等问题.`dumb-init`提供了向子进程代理发送信号和接管子进程的功能从而解决上述问题.

- 制作java基础镜像

**Dockerfile**
```bash
FROM baseimage-alpine:v3.4
MAINTAINER Jeffrey <ji459705636@163.com>

ENV LANG C.UTF-8

# install jdk 8
RUN { \
		echo '#!/bin/sh'; \
		echo 'set -e'; \
		echo; \
		echo 'dirname "$(dirname "$(readlink -f "$(which javac || which java)")")"'; \
	} > /usr/local/bin/docker-java-home \
	&& chmod +x /usr/local/bin/docker-java-home
ENV JAVA_HOME /usr/lib/jvm/java-1.8-openjdk
ENV PATH $PATH:/usr/lib/jvm/java-1.8-openjdk/jre/bin:/usr/lib/jvm/java-1.8-openjdk/bin
ENV JAVA_VERSION 8u111
ENV JAVA_ALPINE_VERSION 8.111.14-r0

RUN set -x \
	&& apk add --no-cache \
		openjdk8="$JAVA_ALPINE_VERSION" \
	&& [ "$JAVA_HOME" = "$(docker-java-home)" ]

CMD ["/bin/sh"]
```

```bash
docker build -t baseimage-java:v8 .
```

&emsp;&emsp;在这个镜像中我们只是在`alpine`基础镜像上安装了java环境.

- 制作具体应用实例镜像

**Dockerfile.tmplt**
```bash
FROM baseimage-java:v8
MAINTAINER Jeffrey <ji459705636@163.com>

RUN mkdir /code
ADD SOURCE.jar /code/source.jar
ENTRYPOINT ["/usr/bin/dumb-init", "--"]
CMD ["java","-jar","/code/source.jar"]
```

&emsp;&emsp;为了可以方便的复用这个`Dockerfile`模板文件,灵活地将`SOURCE.jar`替换成真实的jar名称,然后构建镜像,运行实例,可以使用`Makefile`来编写启动脚本.

**Makefile**
```bash
REGISTRY = 127.0.0.1
VERSION = v1.0.0
APP_NAME = test

build: Dockerfile.tmplt
	    @sed "s/SOURCE/$(APP_NAME)/g" Dockerfile.tmplt > Dockerfile
        docker build -t $(REGISTRY)/$(APP_NAME):$(VERSION) .
        @rm -v Dockerfile
        docker push $(REGISTRY)/$(APP_NAME):$(VERSION)

run:
        docker pull $(REGISTRY)/$(APP_NAME):$(VERSION)
        -docker rm -f $(APP_NAME)
        docker run -d --net=host \
                --name $(APP_NAME) \
                --user root \
                $(REGISTRY)/$(APP_NAME):$(VERSION) \
        docker logs -f $(APP_NAME)
```

&emsp;&emsp;然后我们就可以通过`Makefile`来动态制作镜像(`make build`)和运行容器(`make run`).

### 容器常用启动参数介绍(`docker run`)

- `--net=host`:以主机网络模式启动容器,即容器的端口映射到宿主机相同的端口.默认网络模式为`bridge`模式.
- `-d`:后台模式运行容器.
- `-it`:交互模式运行容器,与`-d`互斥.
- `-p`:端口映射,在`bridge`模式下将容器端口映射到宿主机端口.
- `-e`:设置容器启动环境变量.
- `-v`:设置容器挂载路径.
- `--name`:设置容器名称.
- `--user`:设置容器启动用户.
- `-l`:设置容器标签.
- `--restart=always`:设置容器运行策略,默认为`no`.
- `--rm=true`:设置容器自动删除策略,默认为`false`.

&emsp;&emsp;常用的启动参数大概就这些,更详细的参数可以`man docker run`查看.

### 推送镜像仓库

&emsp;&emsp;镜像仓库顾名思义,就是存放镜像的仓库.当我们`docker search/pull [image_name]`时,都是从官方公共仓库上查询/拉取镜像.我们也可以配置自己的镜像仓库,这个步骤很简单,拉取镜像仓库的镜像,然后启动容器即可.
- `docker pull registry:2`
- `docker run -d -p 5000:5000 --restart=always --name=registry registry:2`

&emsp;&emsp;如果需要把本地镜像(假设名称为:test:v1)推送到远程镜像仓库(假设地址为:172.24.4.1:5000),需要进行如下步骤:
- `docker tag test:v1 172.24.4.1:5000/test:v1`
- `docker push 172.24.4.1:5000/test:v1`

&emsp;&emsp;然后我们就可以其他主机上执行`docker pull 172.24.4.1:5000/test:v1`拉取该镜像了,此时从私有镜像仓库中拉取镜像可能会报错,这是由于本地并不信任远程镜像仓库,此时需要配置信任源.执行如下脚本即信任任何源,当然也可以指定具体的信任源.
```bash
#!/bin/sh

echo '{ "insecure-registries":["0.0.0.0/0"] }' > /etc/docker/daemon.json
systemctl restart docker
```

### 调试小技巧

&emsp;&emsp;很多时候,我们`docker run`启动容器时会由于各种错误导致容器起不来,比如文件没权限,镜像制作的有问题等各种各样的问题.这时我们再重新制作镜像,重新启动容器非常麻烦且很浪费时间.所以我建议可以直接通过`docker run -it --rm (--user=root) [image_name] /bin/sh`以交互模式先启动容器.然后各种调试启动命令,直到解决所有的问题,达到满意的效果,再去编写启动脚本.

### end

&emsp;&emsp;docker确实是个很棒的运维工具,虽然我们不是运维,但是熟悉了docker的使用,可以极大地提高我们开发测试效率.这篇文章只是简单介绍下自己的使用心得,我也没有刻意地去了解的很深,毕竟要学的东西实在太多太多,能够熟练地使用足矣.