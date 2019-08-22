---
title: 使用Docker-compose 搭建LNMP 环境
date: 2019-08-22 17:55:30
tags: docker
---

- Docker 安装请参考官方文档[https://docs.docker.com/install/linux/docker-ee/centos/](https://docs.docker.com/install/linux/docker-ee/centos/)
- Docker Compose 安装请参考官方文档
  [https://docs.docker.com/compose/install/](https://docs.docker.com/compose/install/)

## 一、添加 nginx 配置 Dockerfile

我常用的方式是在项目的代码根目录创建 nginx 目录用作存储 nginx 的配置文件，以及 dockerfile 文件，nginx 配置文件 default.conf 存放在 nginx/conf.d/default.conf。
default.conf 配置文件配置 nginx 域名映射，后端网关代理，地址转发规则等，在用 docker-compose 启动 nginx 时把自定义的配置文件挂载到 nginx 容器配置文件目录里面，覆盖默认配置文件。文件实例如下：

```
server {
    listen 80;
    server_name localhost;
    root /var/www/html/webroot;
    index index.php;
    location / {
        try_files $uri $uri/ /index.php?$args;
    }
    location = /50x.html {
        root /var/www/html;
    }
    location ~ \.php$ {
        root /var/www/html/webroot;
        # 代码目录
        fastcgi_pass phpfpm:9000;
        # 修改为phpfpm容器
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME  $document_root$fastcgi_script_name;
        # 修改为$document_root
        include fastcgi_params;
    }
    #error_page  404              /404.html;
    # redirect server error pages to the static page /50x.html
    #
    error_page 500 502 503 504  /50x.html;
}

```

nginx 的 dockerfile 文件存放在 nginx/Dockerfile,内容如下:

```
FROM nginx:latest
# 这里使用的是nginx的官方镜像源
#COPY conf/default.conf /etc/nginx/conf.d/default.conf
CMD ["nginx", "-g", "daemon off;"]

```

## 二、配置 PHP 的 Dockerfile

我这里使用的是 php5.6-fpm 的 docker 镜像,如果有 php 版本需求，可以自行修改 php7 的镜像。使用 docker-php-ext-install 安装我们程序所需要的扩展包。可以自定义设置 PHP CGI 的监听端口。
示例如下：

```
FROM php:5.6-fpm

RUN requirements="libmcrypt-dev g++ libicu-dev" \
  && apt-get update && apt-get install -y $requirements && rm -rf /var/lib/apt/lists/* \
  && docker-php-ext-install pdo_mysql \
  && docker-php-ext-install mcrypt \
  && docker-php-ext-install mbstring \
  && docker-php-ext-install intl \
  && docker-php-ext-install pcntl \
  && requirementsToRemove="g++" \
  && apt-get purge --auto-remove -y $requirementsToRemove

RUN sed -i -e 's/listen.*/listen = 0.0.0.0:9000/' /usr/local/etc/php-fpm.conf

# RUN usermod -u 1000 www-data

# COPY docker-entrypoint.sh /entrypoint.sh
# ENTRYPOINT ["/entrypoint.sh"]

CMD ["php-fpm"]
```

## 三、配置 docker-compose.yaml 文件

一般情况下，会把 docker-compose.yaml 文件放到我们的程序根目录下面。方便我们启动环境。
Docker-compose.yaml 文件示例如下:

```
version: '3.3'

services:
  db:
    image: mysql:5.6
    volumes:
      - db_data:/var/lib/mysql
    restart: always
    ports:
      - '3306:3306'
    environment:
      MYSQL_ROOT_PASSWORD: Chinamcloud@2018
      MYSQL_DATABASE: cmop
      MYSQL_USER: cmop
      MYSQL_PASSWORD: newmedia

  phpfpm:
    # depends_on:
    #   - db
    build: ./phpfpm
    ports:
      - '9000:9000'
    restart: always
    volumes:
      # 挂载当前程序到容器的目录里面，放到php解析目录
      - ./:/var/www/html/
    links:
      - 'db'
    # environment:
    #   WORDPRESS_DB_HOST: db:3306
    #   WORDPRESS_DB_USER: wordpress
    #   WORDPRESS_DB_PASSWORD: wordpress
  nginx:
    build: ./nginx
    ports:
      - '80:80'
    links:
      - 'phpfpm'
    volumes:
      # 把当前程序文件挂载到nginx容器里面
      - ./:/var/www/html/
      # 把当前的自定义配置文件挂载到nginx容器里面
      - ./nginx/conf/default.conf:/etc/nginx/conf.d/default.conf

  # 这里配置了一个调用第三方的API服务
  api:
    build: ./tools_go/api
    links:
      - 'db'
    restart: always
    ports:
      - '9090:9090'
volumes:
  db_data: {}

```

在这个配置文件里面，mysql 使用的方式是直接获取官方镜像启动，nginx 和 php-fpm 使用的是本地 Dockerfile build 出来的镜像做为容器基础镜像。同时 MYSQL 设置了环境变量，数据库的账号密码等信息。

## 四、启动 docker-compose

- 启动程序 docker-compose up
- 关闭程序 docker-compose down

启动完程序，你可以通过你配置的 nginx 的域名访问服务了。更多 docker-compose 命令请参考官方文档[https://docs.docker.com/compose/reference/overview/](https://docs.docker.com/compose/reference/overview/)
