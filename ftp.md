# ftp

## 1、 vsftp在centos7 配置

##1.1、安装

```
 vsftpd -v   # 查看是否已安装
 sudo yum -y install vsftpd   #安装ftp
 service vsftpd start      #启动vsftpd服务
```

## 1.2、配置

 ```
vi /etc/vsftpd/user_list和vi /etc/vsftpd/ftpusers两个设置文件脚本，将root账户前加上#号变为注释。（即让root账户从禁止登录的用户列表中排除）
vsftpd.userlist中明确列出的用户才能被允许登录
/etc/vsftpd/ftpusers   拒绝登录的user列表
 ```

## 1.3、连接

协议：FTP-文件传输协议

ip：54.183.236.104

端口：21

用户名：root

密码：服务器密码123

