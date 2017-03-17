`使用consul+consul_template+registartor+docker进行动态管理nginx的upstream后端server:  ` 

* 基本原理就是：     
　　使用consul做服务发现（进行服务的注册类似etcd），使用consul_template(类似confd)进行nginx配置文件的更新，使用registartor进行检测docker上所起的服务并注册到指定的consul机器上．



* 快速搭建：

一．consul的搭建   
=====
　　前面文章讲过consul的单机以及集群的搭建，下面不在详细介绍相关参数了：     

　　　　docker run -d -p 8500:8500  -p 53:53/udp --name=consul  gliderlabs/consul-server -bootstrap　-advertise 127.0.0.1         
　　　　
　　　　注意:-advertise参数表示的是:是给集群中的其他节点展现的地址.      

二．registartor的搭建   
====

    docker run -d \
    --name=registrator \
    --volume=/var/run/docker.sock:/tmp/docker.sock \
    gliderlabs/registrator:latest \
    consul://localhost:8500
　简单介绍下上面参数的含义：    
　　　　第三行表示把docker的socket文件挂载到该docker容器下　第五行表示当检测到docket上有新的服务的时候　会把其注册到指定的consul            

   注意:此时是需要把localhost换成你搭建consul的那台机器的真实ip(对外出口ip),由于其docker默认使用的 bridge网络模式.        


三．consul_template与nginx的搭建   
===
   下面把其consul_template和nginx进行了dockerfile:
   使用docker build -t drname  .  进行生成镜像文件   
　　然后执行进行文件：    
　　　　docker run -it \
　　　　-e "CONSUL=$DOCKER_IP:8500" \
　　　　-e "SERVICE=test" \
　　　　-p 80:80 drname
上面表示当有名字为test服务的时候　此时就会把该test服务的server加载到nginx的upstream中去．     
-e参数表示给定docker中的变量赋值    
-p表示端口的映射    
-it是交互式  

注意本地的nginx.conf upstream.conf的编写，关于nginx相关讲解可以看前面文章．



Dockerfile:         

``` nginx         

#tar -C /usr/local/bin --strip-components 1 -zxf -
ADD consul-template /usr/local/bin/

#Setup Consul Template Files
RUN mkdir /etc/consul-templates
ENV CT_FILE /etc/consul-templates/nginx.conf

#Setup Nginx File
ENV NX_FILE /etc/nginx/conf.d/app.conf

#Default Variables
ENV CONSUL consul:8500
ENV SERVICE consul-8500

# Command will
# 1. Write Consul Template File
# 2. Start Nginx
# 3. Start Consul Template
#ADD nginx.conf /etc/nginx/nginx.conf

CMD echo "events { worker_connections 100; }     \n\ 
  http {                                 \n\
  upstream app {                         \n\
  least_conn;                            \n\
  {{range service \"$SERVICE\"}}         \n\
  server  {{.Address}}:{{.Port}};        \n\
  {{else}}server 127.0.0.1:65535;{{end}} \n\ 
}                                        \n\
server {                                 \n\
  listen 81 default_server;              \n\
  location / {                           \n\
    proxy_pass http://app;               \n\
  }                                      \n\
} }" > $CT_FILE; \
/usr/sbin/nginx -c /etc/nginx/nginx.conf \
& CONSUL_TEMPLATE_LOG=debug consul-template \
  -consul=$CONSUL \
  -template "$CT_FILE:/etc/nginx/nginx.conf:/usr/sbin/nginx -c /etc/nginx/nginx.conf -s reload";
  
```    


四．测试   
====
　　此时已经搭建好了　当我们使用docker进行启动一个名字为test的服务的时候，此时registator会检测到docker的socket有变化则会主动的把该服务注册到指定的consul上．然后consul_template发现指定的consul上的服务有变化的时候,会主动的更新nginx的配置文件，然后重启nginx.

　　如：   
　　　docker run -it \  
　　　-e "SERVICE_NAME=test" \   
　　　-p 8000:80 python/server    



Communite  
====
 
在使用中有任何问题，欢迎反馈给我，可以用以下联系方式跟我交流

* 邮件(1031379296#qq.com, 把#换成@)
* QQ: 1031379296
* weibo: [@王发康](http://weibo.com/u/2786211992/home)


Thx
====

* chunshengsterATgmail.com


Author
====
* Linux\nginx\golang\c\c++爱好者
* 欢迎一起交流  一起学习# 
* Others say good and Others good


