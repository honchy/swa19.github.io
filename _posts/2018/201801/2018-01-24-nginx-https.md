#Nginx做网络出口的代理和转发


#Nginx配置Http和Https
1. 在3-116机器添加ssl证书,我使用了私有的证书,配置完成后firefox浏览器无法访问,通过Google浏览器可以实现不安全的访问,测试环境下只为了验证配置是否正确,所以选用了私有的证书,当然也可以获取免费的安全证书,参考(https://www.startcomca.com/)
私有证书的安装参考:[nginx配置https(免费证书)](http://blog.csdn.net/frankenjoy123/article/details/76187270)
安装步骤如下:
在nginx的安装目录下:`/usr/local/nginx`下新建文件夹:`makedir conf.d`
生成秘钥:`openssl genrsa -des3 -out server.key 2048`
创建服务器证书的申请文件server.csr:`openssl req -new -key server.key -out server.csr`
其中Country Name填CN,Common Name填主机名也可以不填,如果不填浏览器会认为不安全.(例如你以后的url为https://abcd/xxxx….这里就可以填abcd),其他的都可以不填.
创建CA证书:`openssl req -new -x509 -key server.key -out ca.crt -days 3650`
此时,你可以得到一个ca.crt的证书,这个证书用来给自己的证书签名.
创建自当前日期起有效期为期十年的服务器证书server.crt：`openssl x509 -req -days 3650 -in server.csr -CA ca.crt -CAkey server.key -CAcreateserial -out server.crt`
ls你的文件夹,可以看到一共生成了5个文件:`ca.crt  ca.srl  server.crt server.csr server.key`
2. 配置Nginx
~~~
user  nginx;
worker_processes  1;
error_log  logs/error.log;
events {
    worker_connections  1024;
}
http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;
    upstream 116-web {
        server 127.0.0.1:11111;
        upsync 172.16.3.116:8500/v1/kv/upstreams upsync_timeout=6m upsync_interval=500ms upsync_type=consul strong_dependency=off;
        upsync_dump_path /usr/local/nginx/conf/servers/servers_list.conf;
    }
    server {
        listen 443 ssl;   #监听https请求
        listen       80;   #监听http请求
        server_name  localhost;
        ssl on;
        ssl_certificate_key /usr/local/nginx/conf.d/server.key;
        ssl_certificate /usr/local/nginx/conf.d/server.crt;
        ssl_session_timeout  5m;
        ssl_protocols  SSLv2 SSLv3 TLSv1;
        ssl_ciphers  ALL:!ADH:!EXPORT56:RC4+RSA:+HIGH:+MEDIUM:+LOW:+SSLv2:+EXP;
        ssl_prefer_server_ciphers   on;
        location /api{
                             proxy_pass      http://116-web/api;
                             proxy_connect_timeout   1800s;
                             proxy_read_timeout      1800s;
                             proxy_send_timeout 1800s;
                             proxy_buffer_size  16k;
                             proxy_buffers      16 128k;
                             proxy_busy_buffers_size 256k;
                             proxy_temp_file_write_size 256k;
                             proxy_next_upstream     error;
                             proxy_set_header  X-Real-IP  $remote_addr;
                             client_max_body_size       30m;
                             client_body_buffer_size    256k;
         }
         
         location = /upstream_show {
            upstream_show;
         }
         location = /upstream_status {
            stub_status on;
            access_log off;
         }
     }
}

~~~

# 3-117

~~~
user  nginx;
worker_processes  1;
error_log  logs/error.log;
events {
    worker_connections  1024;
}
http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;

    server {
        listen       80;
        server_name  localhost;
        location /api{
                            proxy_pass   https://server.name.com/api;
                             proxy_connect_timeout   1800s;
                             proxy_read_timeout      1800s;
                             proxy_send_timeout 1800s;
                             proxy_buffer_size  16k;
                             proxy_buffers      16 128k;
                             proxy_busy_buffers_size 256k;
                             proxy_temp_file_write_size 256k;
                             proxy_next_upstream     error;
                             proxy_set_header  X-Real-IP  $remote_addr;
                             proxy_set_header        Host server.name.com;
                             client_max_body_size       30m;
                             client_body_buffer_size    256k;
         }         
     }
}
~~~




# 遇到的问题
1. 对https的请求,若3-116不配置ssl,报错:

~~~
This site can’t be reached
172.16.3.116 refused to connect.
Try:

Checking the connection
Checking the proxy and the firewall
ERR_CONNECTION_REFUSED
~~~

2. 对于协议,若nginx配置文件中不明确指定网络协议(upstream和proxy-pass中均不指定http或https),那么以请求的url中为准,否则以Nginx配置文件中指定的为准.
对于现在的配置,在3-117中指定了https,那么将以https协议发送请求

3. 路径的匹配


3. 请求头中Host的指定
[proxy_set_header设置Host为$proxy_host，$host与$local_host的区别](http://blog.csdn.net/a19860903/article/details/49914131)
proxy_set_header允许重新定义或者添加发往后端服务器的请求头。value可以包含文本、变量或者它们的组合。 当且仅当当前配置级别中没有定义proxy_set_header指令时，会从上面的级别继承配置.nginx对于upstream默认使用的是基于IP的转发，因此对于以下配置：
~~~
     upstream backend {  
        server 127.0.0.1:8080;  
    }  
    upstream crmtest {  
        server crmtest.aty.sohuno.com;  
    }  
    server {  
            listen       80;  
            server_name  chuan.aty.sohuno.com;  
            proxy_set_header Host $http_host;  
            proxy_set_header x-forwarded-for  $remote_addr;  
            proxy_buffer_size         64k;  
            proxy_buffers             32 64k;  
            charset utf-8;  
      
            access_log  logs/host.access.log  main;  
            location = /50x.html {  
                root   html;  
            }  
        location / {  
            proxy_pass backend ;  
        }  
              
        location = /customer/straightcustomer/download {  
            proxy_pass http://crmtest;  
            proxy_set_header Host $proxy_host;  
        }  
    }  
~~~

当匹配到/customer/straightcustomer/download时，使用crmtest处理，到upstream就匹配到crmtest.aty.sohuno.com，这里直接转换成IP进行转发了。假如crmtest.aty.sohuno.com是在另一台nginx下配置的，ip为10.22.10.116，则$proxy_host则对应为10.22.10.116。此时相当于设置了Host为10.22.10.116。如果想让Host是crmtest.aty.sohuno.com，则进行如下设置：`proxy_set_header Host crmtest.aty.sohuno.com;`
http1.1协议要求必须加上host，主要是为同一服务器提供两个以上的站点服务。比如www1.fucku.com和www2.fucku.com两个域名IP相同，由同一台服务器支持，服务器可以根据host域，分别提供不同的服务，在客户端看来是两个完全不同的站点。
关于Host的参数,在Nginx配置中,提供了三个变量,分别是:$host,$http_host,$proxy_host
$http_host:客户端的ip地址
$host:在请求包含“Host”请求头时为“Host”字段的值，在请求未携带“Host”请求头时为虚拟主机的主域名(当前location块的servername)
$proxy_host:代理服务器的ip地址

如果不能正确配置的话会导致页面请求404,如果很不幸地请求的ip地址中有多个域名,可能会请求到其他的域名地址去.

