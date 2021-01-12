## docker基本命令

#### 启动容器

docker run

--name  指定容器名称

-v  挂载目录，外部目录:容器内目录  将这两个目录连接在一起
-i --interactive=false， 打开STDIN，用于控制台交互
-d  --detach=false， 指定容器运行于前台还是后台，默认为false
-t  --tty=false， 分配tty设备，该可以支持终端登录，默认为false

-p 指定端口映射 (主机):(容器)

--restart ：重启策略
--restart=on-failure  \- 只有在非0状态退出时才从新启动容器；

--restart=always - 无论退出状态是如何，都重启容器；

ex:

docker run -idt -p 5000:5000 xxx:xx（指定镜像标签） 
docker run -idt -v /home/sy/searchsvr/contain/:/app/content -p 5000:5000 telsearch:0.0.1

启动一个空容器，再后台运行

docker run -d ubuntu:16.04 /bin/sh -c "while true; do echo hello world; sleep 1;done"



#### 进入容器

xxx: 容器id

docker exec -it xxx bash

docker exec -it xxx sh



#### 创建镜像

. 表示当前文件夹包含dockerfile

xx:xx 镜像标签

--no-cache 不适用缓存

ex:

docker build . -t xx:xx --no-cache



#### 给镜像重新打上标签

ex:

docker tag id(镜像id) xx:xx（新标签）
docker tag xx:xx（旧标签） xx:xx（新标签）



#### 停止指定容器

docker stop 容器ID/容器名称  



#### 删除指定容器

必须是停止状态的容器才能删除

docker rm 容器ID/容器名称



#### 删除指定镜像 

必须是当前没有被容器使用的镜像才能删除

docker rmi 镜像名称:镜像标签  



#### 查看当前正在运行的容器

docker ps       

docker ps -a --no-trunc   

docker inspect worker2



#### 查看所有已创建的容器（包含已停止的容器）

docker ps -a



#### 查看当前容器服务内的所有镜像

docker images  



#### 推送到远程仓库

docker tag telsearch:0.0.1 docker.cnns/shiyu/telsearch:0.0.1 
docker push docker.cnns/shiyu/telsearch:0.0.1



#### 容器打包成镜像
docker commit  (容器名称或id) (镜像名称:0.0.1)




#### 查看docker日志

docker logs [OPTIONS] CONTAINER

```shell
Options:
        --details        显示更多的信息
    -f, --follow         跟踪实时日志
        --since string   显示自某个timestamp之后的日志，或相对时间，如42m（即42分钟）
        --tail string    从日志末尾显示多少行日志， 默认是all
    -t, --timestamps     显示时间戳
        --until string   显示自某个timestamp之前的日志，或相对时间，如42m（即42分钟）
```

ex:

查看指定时间后的日志，只显示最后100行：

```shell
$ docker logs -f -t --since="2018-02-08" --tail=100 CONTAINER_ID
```

查看最近30分钟的日志:

```shell
$ docker logs --since 30m CONTAINER_ID
```

查看某时间之后的日志：

```shell
$ docker logs -t --since="2018-02-08T13:23:37" CONTAINER_ID
```

查看某时间段日志：

```shell
$ docker logs -t --since="2018-02-08T13:23:37" --until "2018-02-09T12:23:37" CONTAINER_ID
```



### docker 在已经存在的容器上增加启动参数

```
docker update --restart=always xxx
```



### docker打包镜像文件并且加载

- 打包文件

```
docker save -o 要保存的文件名  要保存的镜像
docker save -o cc.tar bb:v1.0
```

- 加载文件

```
docker load < cc.tar
```

