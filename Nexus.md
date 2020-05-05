# nexus 

## 使用HomeBrew安装

``` 
brew install nexus
```
## 免安装

tar zxvf /Users/norman/mine/nexus-3.18.1-01-mac.tgz 
cd /Users/norman/mine/nexus/nexus-3.18.1-01/bin 
./nexus start
当nexus解压之后会有两个文件夹nexus3.x和sonatype-work, 默认的密码在sonatype-work文件夹下,会有一个admin.password的文件，里面就是默认的密码，复制即可，然后输入用户名：admin，
修改为：123

安装路径为：/usr/local/Cellar/nexus/2.14.10-01

备份路径： /usr/local/var/nexus/storage 

## 修改端口号

/Users/norman/mine/nexus/nexus-3.18.1-01/etc/nexus-default.properties 

## 迁移

地址：http://localhost:8090/

复制：/Users/norman/mine/nexus