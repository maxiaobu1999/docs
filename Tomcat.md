# Tomcat

## 1、安装

[官网下载地址 一定是上面的Core:下面](http://tomcat.apache.org/download-80.cgi)

```
#下载压缩包
wget http://apache.mirrors.lucidnetworks.net/tomcat/tomcat-8/v8.5.42/bin/apache-tomcat-8.5.42.tar.gz
#解压
 tar -zxvf apache-tomcat-8.5.42.tar.gz 
 #删除安装包
 rm -rf apache-tomcat-8.5.42.tar.gz  
 #改名
 sudo mv apache-tomcat-8.5.42 tomcat1

```

## 2、控制

启动
切换到tomcat的bin目录输入启动命令：
 sh startup.sh或者sudo ./startup.sh
关闭
sudo ./shutdown.sh 
ps -ef | grep tomcat  #检查tomcat7是否已经安装

## 配置





 cd tomcat1/bin  进入bin目录

sudo vi catalina.sh  配置 添加以下

```

```

