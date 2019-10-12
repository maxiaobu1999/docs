#Linux命令

su root  #输入a2243116 切换root登录
mkdir malong #创建malong文件夹
sudo passwd root     #修改root密码
chmod 777 -R /var/ftp/xxx    #修改权限
 chmod +x *.sh       #执行权限
mv  XXX  YYY  #重命名并移动x-》y
yum -y install wget  #安装wget
rm -rf apache-tomcat-7.0.57.tar.gz  #删除安装包
pwd  #打印当前路径
less xxx.txt    #显示文件内容  Q退出
echo $JAVA_HOME  显示环境变量路径
yum install -y unzip zip   #安装zip
unzip sdk-tools-linux-xxxxxxx.zip  #解压zip


环境变量
/etc/profile   对用户（不含root）
/etc/environment  对系统 root变量找不到添加
```
JAVA_HOME=/usr/java/jdk
JRE_HOME=$JAVA_HOME/jre
PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin
CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib
//下面这行不加
export JAVA_HOME JRE_HOME PATH CLASSPATH
```

ls不好使
export PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin:/root/bin