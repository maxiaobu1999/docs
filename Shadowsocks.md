# Shadowsocks

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



 I  P	    : 54.183.236.104

 端口	    : 2334

 密码	    : 123456

 加密	    : aes-128-ctr

 协议	    : auth_sha1_v4_compatible

 混淆	    : plain

 设备数限制 : 99

 单线程限速 : 0 KB/S

 端口总限速 : 0 KB/S

 SS    链接 : ss://YWVzLTEyOC1jdHI6MTIzNDU2QDU0LjE4My4yMzYuMTA0OjIzMzQ 

 SS  二维码 : http://doub.pw/qr/qr.php?text=ss://YWVzLTEyOC1jdHI6MTIzNDU2QDU0LjE4My4yMzYuMTA0OjIzMzQ

 SSR   链接 : ssr://NTQuMTgzLjIzNi4xMDQ6MjMzNDphdXRoX3NoYTFfdjQ6YWVzLTEyOC1jdHI6cGxhaW46TVRJek5EVTI 

 SSR 二维码 : http://doub.pw/qr/qr.php?text=ssr://NTQuMTgzLjIzNi4xMDQ6MjMzNDphdXRoX3NoYTFfdjQ6YWVzLTEyOC1jdHI6cGxhaW46TVRJek5EVTI 

 

  提示: 

 在浏览器中，打开二维码链接，就可以看到二维码图片。

 协议和混淆后面的[ _compatible ]，指的是 兼容原版协议/混淆。