#### nginx配置负载均衡

docker容器启动服务:telsearch， telsearchx，nginxrpc

niginx conf文件配置:

upstream @whitelist {
	server telsearch:5000;
	server telsearchx:5000;
}
server {
	listen 5000;
	location / {
		proxy_pass http://@whitelist;
	}
}

代码连接配置:

"WHITE_URL": "http://nginxrpc:5000"



#### nginx docker

docker pull nginx:latest

创建一个项目文件夹$PWD

在$PWD/conf 文件夹中创建 xxx.conf

docker run -d -v $PWD/conf:/etc/nginx/conf.d -v $PWD/logs:/var/log/nginx --name nginx_test --network fdz nginx:latest



#### 热重载

nginx -s reload

docker exec -it <container id> <path_of_nginx> -s reload