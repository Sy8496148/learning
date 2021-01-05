# Git基本命令

### 初始化仓库

- git init
- git init --bare xxx.git   没有工作区,将git仓库与项目源码分离



### 添加远程连接

- git remote add origin xxx.git 
- git remote add prod app@47.115.128.96:projects/nanchang.git  (ssh)



### 添加代码到缓存区

- git add . 
- git add 目录名称



### 将代码提交到本地仓库

- git commit -m "frist commit"



### 将代码推送到远程仓库

- git push origin master 



### 将代码拉到本地

- git pull origin master



### 创建分支

- git branch dev



### 合并分支

- git merge dev

  在master分支上合并dev代码

  git branch master

  git metger dev



### 查看分支

- git branch



### 切换分支

- git checkout dev



### 退回版本

- git log
- git reset --hard [id]



### 设置私人仓库

前提条件主机与服务器建立了ssh连接 （创建密钥：ssh-keygen -t rsa -C shiyu@cnns.net）

1,在服务器初始化一个仓库

```
cd projects
git init --bare nanchang.git
```

2,本地主机将代码推送到刚刚创建的仓库

```
git remote git remote add prod app@47.115.128.96:projects/nanchang.git  (ssh)
git add .
git commit -m "frist commit"
git push prod master
```

3,服务器克隆git，会自动生成一个nanchang的目录

```
git clone projects/nanchang.git nanchang
```

4,进入目录，拉取代码

```
cd nanchang 
git pull
```



  