# MySql

## 1、 安装

### 1.1 amazon 

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





###1.1.1 navicat连接

![屏幕快照 2018-11-17 上午10.19.11.png](https://upload-images.jianshu.io/upload_images/3796207-0779f7128921bf46.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![屏幕快照 2018-11-17 上午10.21.33.png](https://upload-images.jianshu.io/upload_images/3796207-9e304cb846c8fc91.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)





###1.2 tengxun版本配置
------------
| 软件   | 版本    |
| ------ | ------- |
| CentOS | 7.6     |
| MySQL  | 8.0.11  |
| amazon | red hat |

####安装
（Mysql最新8，contos7） wget https://dev.mysql.com/get/mysql80-community-release-el7-1.noarch.rpm
（Mysql最新8，contos6）wget https://dev.mysql.com/get/mysql80-community-release-el7-1.noarch.rpm
（mysql5.7）wget 'https://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm'
sudo rpm -Uvh mysql57-community-release-el7-11.noarch.rpm
yum repolist all | grep mysql
sudo yum install mysql-community-server

####root用户配置
1、systemctl start mysqld   #MySql服务启动
2、查找root临时密码
grep 'temporary password' /var/log/mysqld.log
![屏幕快照 2018-11-17 上午8.54.59.png](https://upload-images.jianshu.io/upload_images/3796207-4293fe4af2026dc2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
2、连接数据库
mysql -uroot -p    #输入临时密码登陆
3、设置简单密码（默认对密码复杂度有要求）

- 5.7以上
set global validate_password.policy=0;
 set global validate_password.mixed_case_count=0;
 set global validate_password.number_count=3;
set global validate_password.special_char_count=0;
set global validate_password.length=3;
- 5.7以下
 set global validate_password_policy=0;
 set global validate_password_mixed_case_count=0;
 set global validate_password_number_count=3;
set global validate_password_special_char_count=0;
set global validate_password_length=3;

4、修改root密码
 SET PASSWORD FOR 'root'@'localhost' = PASSWORD('123');
ALTER USER 'root'@'localhost' IDENTIFIED BY '123';      #8.0用这句
5、root用户远程访问（生产不推荐）（8.0不好使，其他版本没试）

- 选择 mysql 数据库，因为 mysql 数据库中存储了用户信息的 user 表。
use mysql;
- 在 mysql 数据库的 user 表中查看当前 root 用户的相关信息
select host, user, authentication_string, plugin from user; 
查看表格中 root 用户的 host，默认应该显示的 localhost，只支持本地访问，不允许远程访问。
####配置新用户
```
# 连接数据库
> mysql -u root -p 
 
# 切换数据库
> use mysql;
 
# 创建用户(user:用户名; %:任意ip, 也可以指定固定IP，root默认是localhost; 登录密码： MyPassword123!)
> CREATE USER 'malong'@'%' IDENTIFIED BY '123';
 
# 修改user用户的加密方式
> ALTER USER 'malong'@'%' IDENTIFIED WITH mysql_native_password BY '123';
 
授权，默认创建的用户权限是usage, 就是无权限，只能登录而已，
 
> grant all on *.* to 'malong'@'%';
{ 注意:用以上命令授权的用户不能给其它用户授权,如果想让该用户可以授权,用以下命令: 
	GRANT all ON databasename.tablename TO 'username'@'host' WITH GRANT OPTION; }
#刷新使以上操作生效：
>flush privileges;
```
navicat连接
![屏幕快照 2018-11-17 上午10.19.11.png](https://upload-images.jianshu.io/upload_images/3796207-0779f7128921bf46.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![屏幕快照 2018-11-17 上午10.21.33.png](https://upload-images.jianshu.io/upload_images/3796207-9e304cb846c8fc91.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###  1.3、MAC配置

[下载地址dmg安装包](https://dev.mysql.com/downloads/mysql/)

硬盘里software也有

Root：a2243116

配置mysql环境变量

```shell
cd /usr/local/mysql/bin

vim ~/.bash_profile

添加：

\#mysql

PATH=$PATH:/usr/local/mysql/bin
退出

source ~/.bash_profile
```

解决：2059 - Authentication plugin 'caching_sha2_password' cannot be loaded: dlopen(../Frameworks/caching_sha2_password.so, 2): image not found

```mysql
mysql -uroot -p 
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'a2243116';	
```





##2、数据控迁移

导出sql，ftp上传服务器

```
# 登陆mysql
mysql -u malong -p
# 创建同名数据库
create database norman;
```

执行source命令，导入.sql文件

```
source norman.sql
```

## 3、使用 Navicat 来进行数据库之间的迁移

https://blog.csdn.net/xuanjiewu/article/details/86152633