# docker

##1 Docker简介

go语言实现，

##2 Docker 安装

### 3.1镜像相关命令

删除镜像

```
docker rmi 【镜像ID】
```



### 3.2 容器相关命令

#### 3.2.0 查看镜像

```
docker images
```



#### 3.2.1 查看容器

查看正在运行的容器

```
docker ps
```

查看所有容器

```
doocker ps -a
```

查看最后一次运行的容器

```
docker ps -l
```

查看停止的容器

```
docker ps -f status=exited
```

#### 3.2.2创建与启动容器

创建容器命令：docker run

-i ：表示运行容器

-t ：表示容器启动后会进入其命令行。加入这两个参数后，容器创建就能登陆进去。即分配一个伪终端。

-name ： 为创建的容器命名,唯一创建后不能再创建同名。

-v ：表示目录映射关系（前者是宿主机目录，后者是映射到宿主机上的目录），可以使用多个-v做多个目录或文件映射。注意：最好做目录映射，在宿主机上做修改，然后共享到容器上。

-d ：在run后面加上-d参数，则会创建一个守护式容器在后台运行（这样创建容器后不会自动登录容器，如果只加-i -t两个参数，创建后就会自动进入容器）。

-p ：表示端口映射，前者是宿主机端口，后者是容器内的映射端口。可以使用多个-p做多个端口映射

(1)交互式方式创建容器:退出后关闭

```
docker run -it --name=容器名称 镜像名称：标签 /bin/bash

docker run  -it --name=mycentos ansible/centos7-ansible /bin/bash
```

这时我们通过ps命令查看，发现可以看到启动的容器，状态为启动状态

退出当前容器

```
exit
```

(2)守护式方式创建容器：

```
docker run -di --name=容器名称 镜像名称:标签

docker run -di --name=mysentos2 ansible/centos7-ansible 
```

登陆守护式容器方式：

```
docker exec -it 容器名称(或者容器ID) /bin/bash

docker exec -it mysentos2 /bin/bash
```

#### 3.2.3停止与启动容器

停止容器：

```
docker stop 容器名称（或者容器ID）

docker stop mysentos2
```

启动容器：

```
docker start 容器名称（或者容器ID）

docker start mysentos2
```

#### 3.2.4文件拷贝

如果我们需要将文件拷贝到容器内可以使用cp命令

```
docker cp 需要拷贝的文件或目录 容器名称:容器目录

docker cp initrd.img.old mysentos2:/usr/local
```

也可以将文件从容器内拷贝出来

```
docker cp 容器名称:容器目录 需要拷贝的文件或目录

docker cp mysentos2:/usr/local/initrd.img.old initrd2.img.old
```

#### 3.2.5目录挂载

我们可以在创建容器的时候，将宿主机的目录与容器内的目录进行映射，这样我们就可以通过修改宿主机某个目录的文件从而去影响容器。

创建容器添加-v参数后边为 宿主机目录：容器目录，例如：

```
docker run -di -v /usr/local/myhtml:/usr/local/myhtml --name=mycentos3 centos:7
```

如果你共享的是多级的目录，可能会出现权限不足的提示。

这是因为CentOS7中的安全模式selinux把权限禁掉了，我们需要添加参数--privileged=true 来解决挂载的目录没有权限的问题

#### 3.2.6查看容器的IP地址

我们可以通过一下命令查看容器运行的各种数据

```
docker inspect 容器名称(容器ID)

docker inspect mysentos2
```

也可以直接执行下面的命令直接输出IP地址

```
docjer inspect --format='{{.NetworkSettings.IPAddress}}' 容器名称(容器ID)
```

####3.2.7删除容器

```
docker rm 容器名称(容器ID)
```

## 4应用部署

### 4.1MySQL部署

(1)拉取mysql镜像

```
docker pull centos/mysql-57-centos7
```

(2)创建容器

```
docker run  -di --name=tensquare_mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 mysql
```

-p 代表端口映射，格式为 宿主机映射端口：容器运行端口

-e 代表添加环境变量 MYSQL_ROOT_PASSWORD 是root用户的登陆密码

(3)进入mysql容器

```
docker exec -it tensquare_mysql /bin/bash
```

(4)登陆mysql

```
mysql -u -root -p
```

(5)远程登陆mysql

连接宿主机的IP，指定端口为3306

###4.2tomcat部署

(1)拉取镜像

```
docker pull tomcat:7-jre7
```

(2)创建容器

创建容器 -p表示端口映射 -v目录挂载，目录不存在会自动创建

```
docker run -di --name=mytomcat -p 9000:8080 -v /usr/local/tomcat/webapps tomcat:7-jre7
```

### 4.3Nginx部署

(1)拉取镜像

```
docker pull nginx
```

(2)创建Nginx容器

```
docker run -di --name=mynginx -p 80:80 nginx
```

### 4.4Rdis部署

(1)拉取镜像

```
docker pull redis
```

(2)创建容器

```
docker run -di --name=myredis -p 6379:6379 redis
```

###4.5 [Nexus](https://hub.docker.com/r/sonatype/nexus3/)

```
docker search nexus
docker pull sonatype/nexus3
docker run -d -p 8081:8081 --name mynexus -v /Users/maxiaobu/Documents/nexus_data:/nexus-data sonatype/nexus3
docker logs -f mynexus
```

http://localhost:8081/

默认用户名密码：admin/admin123

停止与启动容器

```
docker stop  mynexus
```

删除容器

```
docker ps -a
docker rm 98b7382a8f4d
```

容器保存为镜像

```
docker commit mynexus baidunexus
```

输出镜像文件

```
docker save -o mynginx.tar baidunexus
```

删除老的镜像

```
docker rmi baidunexus
```

从tar中恢复镜像

```
docker load -i mynginx.tar
```

启动容器

```
docker run -d -p 8081:8081 --name mynexus -v /Users/maxiaobu/Documents/nexus_data:/nexus-data baidunexus
```



###4.6 shadowsocks

## 5迁移与备份

### 5.1容器保存为镜像

我们可以通过以下命令将容器保存为镜像

```
docker commit 容器名称 镜像名称

docker commit mynginx mynginx_i
```

### 5.2镜像备份

我们可以通过以下命令将镜像保存为tar文件

-o ：输出文件

```
docker save -o mynginx.tar mynginx_i
```

### 5.3 镜像的恢复与迁移

首先我们先删除掉mynginx_img镜像 然后执行此命令进行恢复

```
docker load -i mynginx.tar
```

-i 输入的文件

执行后再次检查镜像，可以看到镜像已经恢复

##6. DockerFile

### 6.1 什么是Dockerfile

Dockerfile是由一系列命令和参数构成的脚本，这些命令应用与基础镜像并最终创建一个新的镜像。

1、对于开发人员：可以为开发团队提供一个完全一致的开发环境；

2、对于测试人员：可以直接拿开发时所构建的镜像或者通过Dockerfile文件构建一个新的镜像开始工作了；

3、对于运维人员：在部署时，可以实现应用的无缝移植。

### 6.2 常用命令

| 命令                               | 作用                                                         |
| ---------------------------------- | ------------------------------------------------------------ |
| FROM image_name:tag                | 定义了使用哪个基础镜像启动构建流程                           |
| MAINTAINER user_name               | 声明镜像的创建者                                             |
| ENV key value                      | 设置环境变量(可以写多条)                                     |
| RUN command                        | 是Dickerfile的核心部分（可以写多条）                         |
| ADD source_dir/file dest_dir/file  | 将宿主机的文件复制到容器内，如果是一个压缩文件，将会在复制后自动解压 |
| COPY source dir/file dest_dir/file | 和ADD相似，但是如果有压缩文件并不能解压                      |
| WORKKDIR path_dir                  | 设置工作目录                                                 |

### 6.3 使用脚本创建镜像

步骤：

(1)创建目录

```
mkdir -dir -p /usr/localdockerjdk8
```
(2)下载jdk的tar并上传到服务器中的/usr/local/dockerjdk8目录

(3)创建文件Dockerfile    `vi DOckerfile`

```
#基础镜像名称和ID
FROM centos:7
#指定镜像创建者信息
MAINTAINER itcast
#切换工作目录
WORkDIR /usr
#创建目录
RUN mkdir /usr/local/java
#ADD 是相对论经jar，添加压缩包并自动解压
ADD jdk压缩包.tar.gz /usr/local/java/
#配置JAVA环境变量
ENV JAVA_HOME /usr/local/java/jdk1.8.0_171
ENV JRE_HOME $JAVA_HOME/jre
ENV CLASSPATH $JAVA_HOME/bin/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib:$CLASSPATH
ENV PATH $JAVA_HOMR/bin:$PATH

```

(4)执行命令构建镜像

```
docker build -t='jdk1.8' .
```

注意后边的空格和点，不要省略 .表示当前目录

(5)查看镜像是否建立完成

```
docker images
```

## 7 Docker私有仓库

### 7.1 私有仓库搭建与配置

(1)拉取私有仓库镜像(此步省略)

```
docker pull registry
```

(2)启动私有仓库容器

```
docker run -di --name=registry -p 5000:5000 registry
```

(3)打开浏览器输入地址http://192.168.184.141:5000/v2/_catalog 看到`{"repositories":[]}`表示私有仓库搭建成功并且内容为空

(4)修改daemon.json

```
vi /etc/docker/daemon.json
```

添加以下内容，保存退出。

```
{"insecure-registries":["192.168.184.141:5000"]}
```

此步用于让docker信任私有仓库地址

(5)重启docker服务

```
systemctl restart docker 
```

### 7.2 镜像上传至私有仓库

(1)标记此镜像为私有仓库的镜像

```
docker tag jdk1.8 192.168.184.141:5000/jdk1.8
```

(2)上传标记的镜像

```
docker push 192.168.184.141:5000/jdk1.8
```

##8 容器数据持久化

docker 为我们提供了三种不同的方式将数据挂载到容器中：**volume、bind mount、`tmpfs`**。

docker 镜像是以 layer 概念存在的，一层一层的叠加，最终成为我们需要的镜像。但该镜像的每一层都是 `ReadOnly` 只读的。只有在我们运行容器的时候才会创建读写层。文件系统的隔离使得：

- 容器不再运行时，数据将不会持续存在，数据很难从容器中取出。
- 无法在不同主机之间很好的进行数据迁移。
- 数据写入容器的读写层需要内核提供联合文件系统，这会额外的降低性能。

### 8.1 volume 方式

volume 方式是 docker 中数据持久化的最佳方式。

- docker 默认在主机上会有一个特定的区域（`/var/lib/docker/volumes/` Linux），该区域用来存放 volume。
- 非 docker 进程不应该去修改该区域。
- volume 可以通过 `docker volume` 进行管理，如创建、删除等操作。
- volume 在生成的时候如果不指定名称，便会随机生成。

(1)创建一个卷

```
docker volume create nexus_data
```

(2)查看卷列表

```
 docker volume ls
```

(3)查看卷信息

```
docker volume inspect  nexus_data
```

(4)删除卷

```
 docker volume rm  nexus_data
```

