# 使用docker搭建nginx-php-mysql-redis环境

## 镜像的获取
* `docker pull nginx #获取nginx镜像`
* `docker pull php:7.1-fpm #获取php镜像`
* `dokcer pull mysql #获取mysql镜像`
* `docker pull redis #获取redis镜像`

## 启动镜像

* php
 `docker run -d --name=php -v /docker_web/www:/www php:7.1-fpm`
* redis 
`docker run -d --name=redis redis`
* mysql 
`docker run -d --name=mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=root mysql`
* nginx
`docker run -d --name=nginx -p 8880:80 -v /docker_web/www:/www -v /docker_web/nginx/conf.d:/etc/nginx/conf.d --link php:php --link redis:redis --link mysql:mysql nginx `
```
-d #后台守护
-name #容器自定义名称
-p #端口映射
-v #目录挂载 宿主:容器
-link #连接容器 容器名称:自定义别名
-e MYSQL_ROOT_PASSWORD=root #初始root密码
```

## conf.d配置示例
```.conf
server {
    listen       80;
    server_name  localhost;
    root /home; ##注意这是容器中的目录不是挂载到宿主中的目录
    location / {
        index  index.php;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

    location ~ \.php$ {
        #这里已经把php和nginx两个容器的/home挂载到相同的目录了
        fastcgi_pass   php:9000; ##通过link可以使用php来代替php容器的ipaddress，如果想查询容器ip可以通过sudo docker inspect 容器 命令查询
        fastcgi_index  index.php;
        fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
        include        fastcgi_params;
    }
}

```
## php测试
```php
<?php
phpinfo();
?>
```
## 数据库据连接测试
* php数据库连接测试脚本
```php
<?php
//注意要先开启PDO模块
//host中的172.17.0.2是mysql容器的ipAddress，如果需要连接宿主中的数据库需要注意host不可以使用127.0.0.1需要使用宿主机的公网ip，容器中所有的127.0.0.1都是指容器本身而不是宿主
$mysql_conf = array( 'host' => '172.17.0.2:3306', 'db' => 'test', 'db_user' => 'root', 'db_pwd' => 'root', );
$pdo = new PDO("mysql:host=" . $mysql_conf['host'] . ";dbname=" . $mysql_conf['db'], $mysql_conf['db_user'], $mysql_conf['db_pwd']);//创建一个pdo对象 
$pdo->exec("set names 'utf8'"); 
$sql = "select * from states";
$stmt = $pdo->prepare($sql); $stmt->bindValue(1, 'joshua', PDO::PARAM_STR);
$rs = $stmt->execute(); if ($rs) {
            while ($row = $stmt->fetch(PDO::FETCH_ASSOC)) {
                    var_dump($row); 
                }
        }
$pdo = null;//关闭连接 
?>
```
* PDO模块开启

```
docker exec -it php bash #进入容器
cd /usr/local/bin #进入该目录
./docker-php-ext-install pdo_mysql #安装pdo模块
```
* 连接宿主机的数据时请分配好可连接的ip，修改可连接ip方法如下,注意172.17.0.1是容器的ip，而不是宿主机的ip
```
mysql -u root -p

mysql>use mysql;

mysql> GRANT ALL PRIVILEGES ON *.* TO 'root'@'172.17.0.3' IDENTIFIED BY 'password' WITH GRANT OPTION;

mysql> flush privileges;  #刷新数据库

```

## php-fpm容器中使用composer
* composer安装
```
curl -sS https://getcomposer.org/installer | php
mv composer.phar /usr/local/bin/composer
apt-get update
apt-get install composer
```
* 更换国内镜像
`composer config -g repositories.packagist composer https://packagist.phpcomposer.com`
* 安装git模块
`apt-get install git`
* 安装laravel5.4
`composer create-project laravel/laravel blog --prefer-dist "5.4.*"`
## gitlab部署
* [参考gitlab笔记](https://github.com/FYKANG/gitlab_note)
## jenkins部署
* 安装步骤
    * `docker pull jenkinsci/jenkins #拉取jenkins镜像`
    * `docker run -d --privileged -p 9990:8080 -p 50000:50000 --name=jenkins -v /opt/jenkins:/opt/jenkins --link gitlab-ce:gitlab -u root -v $(which docker)r:/bin/docker jenkinsci/jenkins:latest #运行容器`
    * `chown -R 1000 /opt/jenkins #赋予挂载目录权限`
* jenkins的docker插件配置
1. 开启docker的tcp端口
    * 修改 `/etc/sysconfig/docker` 中设置为
        * `OPTIONS='--selinux-enabled -H unix:///var/run/docker.sock -H tcp://0.0.0.0:2375'`
    * 重启docker
        * `systemctl daemon-reload`
        * `systemctl restart docker`
2. 重要目录
    * `/var/jenkins_home/workspace/ #项目更目录`
## 私有register搭建
1. 拉取镜像
    `docker pull registry:2.0`
2. 运行容器
    `docker run -p 5000:5000 registry:2.0`
3. 修改nginx配置用于ssl认证
    ```conf
    upstream docker-registry{
        server 127.0.0.1:5000;
    }
    server {
        listen 443 ssl;
        server_name  tx.ykthink.cn;
        ssl on;
        ssl_certificate /etc/nginx/conf.d/ssl/1_tx.ykthink.cn_bundle.crt;
        ssl_certificate_key /etc/nginx/conf.d/ssl/2_tx.ykthink.cn.key;

    root /home/blog/public;
        location /v2/ {  
        # Do not allow connections from docker 1.5 and earlier  
        # docker pre-1.6.0 did not properly set the user agent on ping, catch "Go *" user agents  
        if ($http_user_agent ~ "^(docker\/1\.(3|4|5(?!\.[0-9]-dev))|Go ).*$" ) {  
            return 404;  
        }  

        # To add basic authentication to v2 use auth_basic setting plus add_header  
        # auth_basic "registry.localhost";  
        # auth_basic_user_file /etc/nginx/conf.d/registry.password;  
        # add_header 'Docker-Distribution-Api-Version' 'registry/2.0' always;  

        proxy_pass                          http://docker-registry;  
        proxy_set_header  Host              $http_host;   # required for docker client's sake  
        proxy_set_header  X-Real-IP         $remote_addr; # pass on real client's IP  
        proxy_set_header  X-Forwarded-For   $proxy_add_x_forwarded_for;  
        proxy_set_header  X-Forwarded-Proto $scheme;  
        proxy_read_timeout                  900;  
        }  
        location / {
        try_files $uri $uri/ /index.php?$query_string;
            index index.php index.html index.htm;
        }

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   /usr/share/nginx/html;
        }

        location ~ \.php$ {
            fastcgi_pass   php:9000;
            fastcgi_index  index.php;
            fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
            include        fastcgi_params;
        }
    }
    server {
        listen       80;
        server_name  127.0.0.1;
        rewrite ^(.*) https://$host$1 permanent;
    }
    ```
    * 参数说明
    ```
    upstream 自定义地址{
        server 服务器ip:端口;
    }

    location /v2/ {  
    if ($http_user_agent ~ "^(docker\/1\.(3|4|5(?!\.[0-9]-dev))|Go ).*$" ) {  
        return 404;  
    }  

    proxy_pass                          http://对应upstream中的自定义地址;  
    proxy_set_header  Host              $http_host;   # required for docker client's sake  
    proxy_set_header  X-Real-IP         $remote_addr; # pass on real client's IP  
    proxy_set_header  X-Forwarded-For   $proxy_add_x_forwarded_for;  
    proxy_set_header  X-Forwarded-Proto $scheme;  
    proxy_read_timeout                  900;  

    } 
    ```
4. ssl测试
    `curl -v https://host`
5. 镜像上传测试
    * 私有库服务器操作
        * 拉取一个测试镜像`docker pull hello-world`
        * 修改测试镜像标记`docker tag hello-world:latest HostNmae/hello-mine:latest`
        * 上传测试镜像到私有库`docker push HostName/hello-mine:latest`
    * 客户端服务器操作
        * 查看私有库镜像 `curl -v -X GET https://HostName/v2/hello-mine/tags/list`
        ```conf
        > GET /v2/hello-mine/tags/list HTTP/1.1
        > User-Agent: curl/7.29.0
        > Host: tx.ykthink.cn:5000
        > Accept: */*
        > 
        < HTTP/1.1 200 OK
        < Content-Type: application/json; charset=utf-8
        < Docker-Distribution-Api-Version: registry/2.0
        < Date: Mon, 01 Oct 2018 05:23:30 GMT
        < Content-Length: 40
        < 
        {"name":"hello-mine","tags":["latest"]}
        ```
    * 说明
        * 参数
        ```
        HostName 私有库域名
        ```
        * nginx的上传大小限制
            * 编辑`vim /etc/nginx/nginx.conf`中的`http`
            ```
            client_max_body_size 700m;
            ```
            * 重启nginx`docker restart nginx`
## shell脚本构建
* 服务端
```
\cp -rf /var/jenkins_home/workspace/jenkinsTest /opt/jenkins/Make_DokcerFile/code
```
* 客户端
```
docker stop mynginx
docker rm mynginx
docker rmi tx.ykthink.cn/mynginx:v2
docker pull tx.ykthink.cn/mynginx:v2
rm -rf /docker_web/www/*
docker run -d -p 8880:80 --link php:php --link redis:redis -v /docker_web/nginx/conf.d:/etc/nginx/conf.d -v /docker_web/www:/www --name=mynginx  tx.ykthink.cn/mynginx:v2
docker cp mynginx:/home/. /docker_web/www/
cp /docker_web/env/.env /docker_web/www/.env
chmod -R 777 /docker_web/www/storage/
docker exec php bash -c "cd /www && composer install"
```