# MySql

## 1、 安装

## 1.1 amazon 

------------
| 软件   | 版本                                              |
| ------ | ------------------------------------------------- |
| CentOS |                                                   |
| MySQL  | 5.5                                               |
| amazon | Amazon Linux AMI 2018.03.0 (HVM), SSD Volume Type |

```
sudo yum install mysql
sudo yum install mysql-server
sudo yum install mysql-devel
#启动mysql服务，设置登录mysql的用户、密码
sudo service mysqld start
/usr/bin/mysqladmin -u root password '你要设置的密码'
#登陆
mysql -u root -p 

#########
刷新使以上flush privileges;操作生效：
#########

#插入mysql新用户
CREATE USER 'malong'@'host' IDENTIFIED BY '123';
#创建新的database  norman
CREATE DATABASE norman DEFAULT CHARACTER SET utf8 COLLATE utf8_general_ci;
#本地用户赋予所有权限
grant all privileges on norman.* to malong@localhost identified by '123';
#给账户开通外网所有权限
grant all privileges on norman.* to malong@'%' identified by '123';
```





## navicat连接

![屏幕快照 2018-11-17 上午10.19.11.png](https://upload-images.jianshu.io/upload_images/3796207-0779f7128921bf46.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![屏幕快照 2018-11-17 上午10.21.33.png](https://upload-images.jianshu.io/upload_images/3796207-9e304cb846c8fc91.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)