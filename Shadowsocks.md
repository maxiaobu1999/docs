# Shadowsocks

su root  #输入a2243116 切换root登录
mkdir malong #创建malong文件夹
sudo passwd root     #修改root密码
chmod 777 -R /var/ftp/xxx    #修改权限

##1、SSR服务器

```
yum -y install wget
wget -N --no-check-certificate https://raw.githubusercontent.com/ToyoDAdoubi/doubi/master/ssr.sh && chmod +x ssr.sh && bash ssr.sh

请输入数字 [1-15]：1
(默认: 2333):2333
请输入要设置的ShadowsocksR账号 密码
(默认: doub.io):123456
(默认: 5. aes-128-ctr):5
(默认: 2. auth_sha1_v4):2
是否设置 协议插件兼容原版(_compatible)？[Y/n]Y
(默认: 1. plain):1
[注意] 设备数限制：每个端口同一时间能链接的客户端数量(多端口模式，每个端口都是独立计算)，建议最少 2个。
(默认: 无限):99
[注意] 单线程限速：每个端口 单线程的限速上限，多线程即无效。
(默认: 无限):回车即默认

```

===================================================



 ShadowsocksR账号 配置信息：



I  P	    : 52.199.159.43

 端口	    : 2333

 密码	    : 123456

 加密	    : aes-128-ctr

 协议	    : auth_sha1_v4_compatible

 混淆	    : plain

 设备数限制 : 0(无限)

 单线程限速 : 0 KB/S

 端口总限速 : 0 KB/S

 SS    链接 : ss://YWVzLTEyOC1jdHI6MTIzNDU2QDUyLjE5OS4xNTkuNDM6MjMzMw 

 SS  二维码 : http://doub.pw/qr/qr.php?text=ss://YWVzLTEyOC1jdHI6MTIzNDU2QDUyLjE5OS4xNTkuNDM6MjMzMw

 SSR   链接 : ssr://NTIuMTk5LjE1OS40MzoyMzMzOmF1dGhfc2hhMV92NDphZXMtMTI4LWN0cjpwbGFpbjpNVEl6TkRVMg 

 SSR 二维码 : http://doub.pw/qr/qr.php?text=ssr://NTIuMTk5LjE1OS40MzoyMzMzOmF1dGhfc2hhMV92NDphZXMtMTI4LWN0cjpwbGFpbjpNVEl6TkRVMg 