---
layout: post
title: Nginx 基础
categories: nginx
tags: nginx
date: 2023-04-25
---
Nginx 的基础笔记
<!--more-->
## 1. 安装 nginx
使用 docker 学习测试 nginx
```bash
#启动，挂载日志文件、配置文件、静态文件
docker run \
    --name nginx80 \
    --network inner-network \
    -v /opt/docker-volume/nginx/logs:/var/log/nginx \
    -v /opt/docker-volume/nginx/conf:/etc/nginx/conf.d:ro \
    -v /opt/docker-volume/nginx/www:/usr/share/nginx/www:ro \
    -p 80:80 \
    -d nginx:1.24

#创建配置文件
echo "server {
    listen       80;
    listen  [::]:80;
    server_name  localhost;

    location / {
        #获取代理之前真实 IP
        #proxy_set_header    Host  \$host;
        #proxy_set_header    X-Real-IP  \$remote_addr;
        #proxy_set_header    X-Forwarded-For  \$proxy_add_x_forwarded_for;
        #proxy_set_header    X-Forwarded-Proto \$scheme;
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}" > /opt/docker-volume/nginx/conf/self.conf

#修改时区
docker exec nginx80 ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
docker exec nginx80 echo 'Asia/Shanghai' > /etc/timezone

#重新加载 nginx 配置文件
docker exec nginx80 nginx -s reload

#宿主机中配置日志分割，可不配置，此项需要安装 logrotate
#日志切割可能会两天分到一天中，可以自己修改 /etc/anacrontab，不修改问题也不大
echo "/opt/docker-volume/nginx/logs/*.log {
        daily
        missingok
        rotate 52
        compress
        delaycompress
        dateext
        notifempty
        dateyesterday
        create 640 root root
        sharedscripts
        postrotate
            docker exec nginx80 bash -c \"if [ -f /var/run/nginx.pid ]; then kill -USR1 `docker exec nginx80 cat /var/run/nginx.pid`; fi\"
        endscript
}" > /etc/logrotate.d/nginx

#测试
curl -I 127.0.0.1
```

Docker 容器中 nginx 的配置目录在 `/etc/nginx` 

`/etc/nginx/nginx.conf`
```
user  nginx;
#工作的进程数，一半与 cpu 核数相关，一个核对应一个工作进程
worker_processes  auto;

error_log  /var/log/nginx/error.log notice;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
    #引入 http mime 类型
    include       /etc/nginx/mime.types;
    #mime类型没匹配上，默认使用二进制流的方式传输
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    #on 数据 0 拷贝，文件不经过 nginx，直接由 linux sendfile(socket, file, len) 高效网络传输
    #off 则文件先由 nginx 读入内存，再由 nginx 发送
    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;

    #读入 /etc/nginx/conf.d/ 目录下所有以 .conf 结尾的配置文件，一个文件可以对应一个 vhost
    include /etc/nginx/conf.d/*.conf;
}
```

`/etc/nginx/conf.d/default.conf `
```
server {
    #监听端口号
    listen       80;
    listen  [::]:80;
    #主机名，可以有多个匹配规则
    server_name  localhost;

    #access_log  /var/log/nginx/host.access.log  main;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

    # proxy the PHP scripts to Apache listening on 127.0.0.1:80
    #
    #location ~ \.php$ {
    #    proxy_pass   http://127.0.0.1;
    #}

    # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
    #
    #location ~ \.php$ {
    #    root           html;
    #    fastcgi_pass   127.0.0.1:9000;
    #    fastcgi_index  index.php;
    #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
    #    include        fastcgi_params;
    #}

    # deny access to .htaccess files, if Apache's document root
    # concurs with nginx's one
    #
    #location ~ /\.ht {
    #    deny  all;
    #}
}
```

## 2.虚拟主机
一个 server 对应一个虚拟主机
### 2.1 sever_name
sever_name，是请求的主机名，如：alamide.com ，常用于多级域名分发请求，示例如下

有 `data.conf`
```
server {
    listen       80;
    listen  [::]:80;
    server_name  data.alamide.com;

    location / {
        root   /usr/share/nginx/www/data;
        index  index.html;#index.html 中内容为 This is data-index.html.
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```

及 `img.conf`
```
server {
    listen       80;
    listen  [::]:80;
    server_name  img.alamide.com;

    location / {
        root   /usr/share/nginx/www/img;
        index  index.html;#index.html 中内容为 This is img-index.html.
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```

修改本地 host 文件
```
127.0.0.1 data.alamide.com

127.0.0.1 img.alamide.com
```

测试
```shell
#nginx 重新加载配置文件
docker exec nginx01 nginx -s reload

#This is data-index.html.
curl data.alamide.com

#This is img-index.html.
curl img.alamide.com
```

不同的子域名映射到两个虚拟主机

server_name 有如下匹配规则，匹配分先后顺序，上面匹配，下面就不会匹配了，.conf 文件会按字母顺序加载
1. 完整匹配，可以在一个 server_name p匹配多个域名
   ```
   server_name data.alamide.com x.alamide.com;
   ```

2. 通配符匹配
   ```
   sever_name *.alamide.com

   server_name alamide.*
   ```

3. 正则匹配
   ```
   # ~ 表示大小写敏感，
   server_name ~^[a-z]+\.alamide\.com$;
   ```

### 2.2 反向代理
可以将请求转发，可以把请求转发到 Tomcat 或其它服务器，再把请求的结果返回，下面请求转发到 百度
```
server {
    listen       80;
    listen  [::]:80;
    server_name  baidu.alamide.com;

    location / {
        proxy_pass http://www.baidu.com;

        #转发到 tomcat
        #proxy_pass http://localhost:8080;
    }
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```

修改本地 hosts 文件
```
127.0.0.1 baidu.alamide.com
```

测试
```shell
#重新加载配置文件
docker exec nginx01 nginx -s reload
curl baidu.alamide.com
```

可以看到返回百度首页内容

## 3.负载均衡
业务繁忙时会把一个服务放置在多台机器上，这时不能出现一台过于繁忙，而另一台机器很闲的情况，可以通过 nginx 来达到负载均衡

`data.conf`

```
#轮询，缺点是无法保持会话，因为 session 信息是由 Tomcat 保存的，而轮询不能保证整个请求周期访问同一台服务器
#会话保持可以 下发 token、SpringSession
upstream httpd {
   server 192.168.1.5:8001;
   server 192.168.1.5:8002;
   server 192.168.1.5:8003;
}

#权重
upstream httpd_weight {
   server 192.168.1.5:8001 weight=10;
   server 192.168.1.5:8002 weight=30;
   server 192.168.1.5:8003 weight=50;
}

#down 表示不参与负载，backup 表示其它所有的非backup机器down或者忙的时候，请求backup机器。
upstream httpd_other {
   server 192.168.1.5:8001 weight=10 down;
   server 192.168.1.5:8002 weight=30;
   server 192.168.1.5:8003 weight=50 backup;
}

server {
    listen       80;
    listen  [::]:80;
    server_name  localhost;

    location / {
        proxy_pass http://httpd;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```

## 4.动静分离
一个复杂的页面可能含有上百个静态资源，如 css、js、image 等，访问的流程是 浏览器 <--> Nginx <--> Tomcat ，如果把静态资源放在 Tomcat 服务器
会使 Tomcat 负担过重，可以把静态资源放在 Nginx 上

```
server {
    
    listen       80;
    listen  [::]:80;
    server_name  localhost;

    #access_log  /var/log/nginx/host.access.log  main;

    location / {
        proxy_pass http://192.168.1.5:8004;        
        #root   /usr/share/nginx/html;
        #index  index.html index.htm;
    }

   #http://124.221.108.130/img/1.jpeg 会去加载 /usr/share/nginx/www/static/img/1.jpeg
   location /img {
        root /usr/share/nginx/www/static;
        index index.html;
   }

   location /css {
        root /usr/share/nginx/www/static;
        index index.html;
   }

   location /js {
        root /usr/share/nginx/www/static;
        index index.html;
   }
}
```

location 匹配规则，“=”匹配 > “^~”匹配 > 正则匹配 > 普通
1. `/ 通用匹配，任何请求都会匹配到`

2. `= 精准匹配 location = /50x.html`

3. `～ 正则匹配，区分大小写 location = ~/(css|img|js)`

4. `~* 正则匹配，不区分大小写  location = ~*/(css|img|js)`

5. `^~ 非正则匹配，匹配以指定模式开头的location`

```
location ~/(css|img|js) {
    proxy_pass http://192.168.1.5:8004;        
    #root   /usr/share/nginx/html;
    #index  index.html index.htm;
}

location /img {
    root /usr/share/nginx/www/static;
    index index.html;
}
```

会返回 Tomcat 而不是 Nginx 的静态资源

## 5.UrlRewrite
可以重写请求的 url，有时候我们不想暴露具体的接口细节，会有这样的请求 .../10000.html，其实最后访问的是
 .../page?page=10000 ，可以使用 nginx 的 UrlRewrite 来转换

规则：rewrite <regex> <replacement> [flag];

flag: 
1. permanent 永久重定向，浏览器地址会显示跳转后的 url

2. break 匹配完成终止，不再向后匹配
```
location / {
    #http://127.0.0.1/10000.html ---> http://127.0.0.1/page?page=10000
    rewrite ^/([0-9]+).html$ /page?page=$1 break;
    proxy_pass http://192.168.1.5:8005;
}
```

还可以实现域名转换，公司旧域名有业务需求变更，需要使用新域名代替，但是旧域名不能废除，需要跳转到新域名上，而且后面的参数保持不变。
```
location / {
    rewrite ^/(.*)$ http://www.newsite.com/$1 permanent;
}
```

## 6.防盗链
语法：valid_referers none | blocked | server_names | string ...

none: 访问的请求头中不带有 referer 时，可访问图片

server_names: 允许访问访问请求头中的 referer 地址
```
location /img {
    valid_referers none 127.0.0.1;
    if ($invalid_referer){
        return 500;
    }
    root /usr/share/nginx/www/static;
    index index.html;
}
```

重写，返回错误图片
```
location /img {
    valid_referers none 127.0.0.1;
    if ($invalid_referer){
        rewrite ^/  /img/x.jpeg break;
    }
    root /usr/share/nginx/www/static;
    index index.html;
}
```

## 7. Https 证书配置
最初的信息传输是明文传输，信息在网络上传输要经过很多个路由器交换机，明文传输是非常不安全的，向服务器提交的数据很大可能被恶意读取并篡改。

先去正规厂商申请 CA 证书，比如阿里、腾讯，再如下配置就可以了
```
server {
    listen       443 ssl;
    server_name  xxx.com;

    # 下面两个文件为申请的公钥和私钥
    ssl_certificate /etc/nginx/conf.d/ssl.pem;
    ssl_certificate_key /etc/nginx/conf.d/ssl.key;

    # ssl验证相关配置
    ssl_session_timeout  5m;    #缓存有效期
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;    #加密算法
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;    #安全链接可选的加密协议
    ssl_prefer_server_ciphers on;   #使用服务器端的首选算法
    
    location / {    
        proxy_set_header    Host  $host;
        proxy_set_header    X-Real-IP  $remote_addr;
        proxy_set_header    X-Forwarded-For  $proxy_add_x_forwarded_for;
        proxy_set_header    X-Forwarded-Proto $scheme;
        proxy_pass          http://server8080:8080;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}

server {
     listen 80;
     server_name data.alamide.com;
     rewrite ^(.*) https://$server_name$1 permanent;
}
```