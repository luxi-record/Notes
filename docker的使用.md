<center><font face="黑体" size=24 >前端关于docker的一些使用</font></center>


##### docker通常在前端可以帮助我们快速的部署项目
###### docker镜像和容器多的区别：可以把docker容器理解为一个镜像的实例，每一个容器都是基于一个镜像启动的，容器内可以理解为一个单独的运行了某个镜像的机器。

* 一些docker常用的命令
  
  1.docker images  查看拉取的镜像
  
  2.docker pull nginx:latest  拉取nginx镜像最近一个版本
  
  3.docker ps -a  查看启动的容器， -a 代表包含那些启动后终止的容器
  
  4.docker run -p 80:80 --name nginx-cont -d -v /static:/etc/nginx/static /conf:/etc/nginx/conf nginx:latest  以nginx最近一个版本镜像启动一个nginx容器。（-p 80:80 将容器的80端口映射到宿主机的80端口）（--name 容器的名字）（-d 表示后台运行）（-v /static:/etc/nginx/static 表示把宿主机的static挂在到容器的/etc/nginx/static目录下）

  5.docker exec -it name /bin/bash  进入name容器内部，有些可能没有bash需要把命令换成 docker exec -it name /bin/sh，进入容器内部后就可以执行各种linux命令

  6.docker logs name  查看name容器的日志

  7.docker stop name｜id  停止运行name这个容器，name可以换成id

  8.docker rm name｜id  删除name这个容器，name可以换成id

  9.docker restart name｜id  重启name容器，name可换成id

  10.docker build -t image .  根据项目的dockerfile构建image镜像

  11.docker rmi image  删除本地image镜像

  12.docker push images  推送镜像到仓库



  ```javascript
  // 一个基本的前端基于nginx镜像的dockerfile
  # 首先，我们需要一个Node环境来构建我们的前端项目
  FROM node:latest  as build-stage

  # 设置工作目录
  WORKDIR /app
  
  # 复制package.json和package-lock.json
  COPY package*.json ./
  
  # 安装项目依赖
  RUN npm install
  
  # 复制项目文件
  COPY . .
  
  # 构建项目
  RUN npm run build
  
  # 然后，我们需要一个Nginx环境来部署我们的前端项目
  FROM nginx:stable-alpine as production-stage
  
  # 从构建阶段复制构建结果到Nginx指定的目录
  COPY --from=build-stage /app/dist /usr/share/nginx/html
  
  # 暴露80端口
  EXPOSE 80
  
  # 启动Nginx
  CMD ["nginx", "-g", "daemon off;"]
  ```
  
