# java

# open jdk 卸载

查找OpenJDK安装包

```perl
rpm -qa | grep openjdk
```



卸载OpenJDK安装包

yum -y remove java-1.7.0-openjdk-headless.x86_64

# 安装

#### 1.[官网下载tar]( https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)

通过ftp上传到服务器

```
[root@ip-172-31-7-35 java]# pwd
/norman/java
#解压
tar -zxvf  jdk-8u211-linux-x64.tar.gz
#改名
mv jdk1.8.0_211 jdk
#配置环境变量
vi /etc/profile
#添加变量
JAVA_HOME=/norman/java/jdk
JRE_HOME=$JAVA_HOME/jre
PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin
CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib
export JAVA_HOME JRE_HOME PATH CLASSPATH
#使修改生效
source /etc/profile 
```



http://54.183.236.104:8088/mini/queryVideo?videoNum=6