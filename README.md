# 使用docker搭建nginx-php-mysql-redis环境

## 镜像的获取
`docker pull nginx #获取nginx镜像`
`docker pull bitnami/php-fpm #获取php镜像`
`dokcer pull mysql #获取mysql镜像`
`docker pull redis #获取redis镜像`

## 启动镜像

* php
 `docker run -d --name=php -v /docker_web/www:/home bitnami/php-fpm`
* redis 
`docker run -d --name=redis redis`
* mysql 
`docker run -d --name=mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=root mysql`
* nginx
`docker run -d --name=nginx -p 8880:80 -v /docker_web/www:/home -v /docker_web/nginx/conf.d:/etc/nginx/conf.d --link php:php --link redis:redis --link mysql:mysql nginx `
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
##数据连接测试

```php
<?php
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