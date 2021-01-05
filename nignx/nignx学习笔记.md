### Nginx 安装

1，下载安装包

wget 下载链接

2，解压安装包

tar -xzf 安装包名称

3，编译安装到指定目录

进入目录解压目录

 ./configure --prefix=/home/geek/nginx

make 

make install



### Nginx 版本升级

1，查看nginx进程

ps -ef | grep nginx

2，备份旧二进制文件 sbin 目录

cp nginx nginx.old

3，复制新文件覆盖旧的二进制文件

cp  -r nginx xxxx/sbin/ -f

4，给旧的nginx进程发送热备份信号

kill -USR2 PID

5，停止旧的nginx work进程

kill -WINCH PID



### 日志切割

1，mv xxx.log bak.log

2,  /sbin/nginx -s reopen



