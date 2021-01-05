- 在官网找到对应的版本的下载链接 https://www.mongodb.com/try/download/community

- 下载

  ```
  wget https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-ubuntu1804-4.2.8.tgz
  ```

- 解压

  ```
  tar -zxvf mongodb-linux-x86_64-ubuntu1604-4.2.8.tgz
  ```

- 将解压后的文件拷贝到指定的目录

  ```
  mv mongodb-linux-x86_64-ubuntu1604-4.2.8  /usr/local/mongodb
  ```

- 添加到路径

  ```
  export PATH=/usr/local/mongodb/bin:$PATH
  ```

- 创建启动配置文件

  ```
  # 老版本
  bind_ip=0.0.0.0
  dbpath=/home/data/mongod/db
  logpath=/home/data/mongod/log/logs.log
  logappend=true
  port=27017
  fork=true
  #auth=true
  
  # 新版本
  systemLog:
     destination: file
     path: "/home/data/mongod/log/logs.log"
     logAppend: false
  storage:
     dbPath: "/home/data/mongod/db"
  net:
     bindIp: 0.0.0.0
     port: 27017
  security:
      authorization: enabled
  processManagement:
     fork: true
  ```

- 创建db路径

  ```
  mkdir -p /home/data/mongod/db
  ```

- 创建log路径，以及文件

  ```
  mkdir -p /home/data/mongod/log/
  touch -p /home/data/mongod/log/logs.log
  ```

- 启动mongdb服务 (bin目录)

  ```
  ./mongod -f mongodb.conf
  ```

- 设置账号密码，mongodb密码和传统数据如mysql等有些区别： mongodb的用户名和密码是基于特定数据库的，而不是基于整个系统的。所有所有数据库db都需要设置密码。

  设置workbench数据库的账号密码  (bin目录)

  ```
  # 关闭权限验证进入启动
  创建管理员账号
  ./mongo # 进入shell
  
  use admin # 切换到admin数据库
  
  db.createUser(
  {
      user: "admin",
      pwd: "admin",
      roles: [
          {role: "root", db: "admin"} 
      ]
  }) # 创建管理员
  
  # 开启权限后重启服务
  ./mongo # 进入shell
  
  use admin # 切换到admin数据库
  
  db.auth('admin','admin') # 登录管理员
  
  use workbench #切换到该数据库
  
  db.createUser(
  { 
      user: "cnns",
      pwd: "cnns.net",
      roles: [
      "dbAdmin", "readWrite"
      ]
  }
  )
  
  use screen
  db.createUser(
  { 
      user: "cnns",
      pwd: "cnns.net",
      roles: [
      "dbAdmin", "readWrite"
      ]
  }
  )
  
  
  # 创建管理该数据库的用户
  
  # Read：允许用户读取指定数据库
  # readWrite：允许用户读写指定数据库
  # dbAdmin：允许用户在指定数据库中执行管理函数，如索引创建、删除，查看统计或访问system.profile
  # userAdmin：允许用户向system.users集合写入，可以找指定数据库里创建、删除和管理用户
  ```

- Ubuntu开放端口

  sudo ufw allow 27017

  sudo ufw status

  sudo ufw reload

- mongodb作为服务启动

  ```
  cd /lib/systemd/system 
  
  sudo vim mongodb.service
  
  [Unit]  
  Description=mongodb  
  After=network.target remote-fs.target nss-lookup.target  
    
  [Service]  
  Type=forking  
  ExecStart=/usr/local/mongodb/bin/mongod --config /usr/local/mongodb/bin/mongodb.conf  
  ExecReload=/bin/kill -s HUP $MAINPID  
  ExecStop=/usr/local/mongodb/bin/mongod --shutdown --config /usr/local/mongodb/bin/mongodb.conf  
  PrivateTmp=true  
    
  [Install]  
  WantedBy=multi-user.target
  
  chmod 754 mongodb.service
  
  #启动服务  
  systemctl start mongodb.service  
  #关闭服务  
  systemctl stop mongodb.service  
  #开机启动  
  systemctl enable mongodb.service
  ```

- 几个查看服务端运行的命令

  netstat -lanp | grep "27017"

  pstree -p | grep mongod

  find / -name "*.log" | xargs grep "vl" 

#### docker 部署

docker pull mongo:latest

docker run -p 27017:27017 -v /data/mongo:/data/db --name mongo -d mongo:latest

