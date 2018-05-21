![image](https://img.shields.io/badge/test-passing-yellowgreen.svg)

# nginx-zuul-dynamic-lb
:maple_leaf: 基于Lua的Spring Cloud网关高可用通用Ngnix插件

***

## 场景痛点

![image](https://github.com/SpringCloud/eureka-admin/blob/master/eureka-admin-sample/eureka-admin-sample-eureka-server/img/dynamic-eureka-server.png)

在Spring Cloud微服务架构体系中，我们往往会部署一个Zuul集群来横向扩展我们的微服务应用，集群的上层是Nginx软负载，在实际情况中，往往会遇到Zuul宕机的尴尬事情，这时候从Nginx到这台机器的请求就会全部失效。此项目针对此痛点，用lua脚本实现定时拉取特定服务地址，动态无感知增减Zuul在Nginx中的负载节点。

如果您希望实现从Nginx直接到普通服务的动态节点负载，在下文配置服务名与Eureka注册中心地址即可。

## OpenResty安装与配置
```
1、环境
yum -y install readline-devel pcre-devel openssl-devel gcc
2、下载解压OpenResty包
wget https://openresty.org/download/openresty-1.13.6.1.tar.gz
tar -zxvf openresty-1.13.6.1.tar.gz
3、下载ngx_cache_purge模块，该模块用于清理nginx缓存
cd openresty-1.13.6.1/bundle
wget http://labs.frickle.com/files/ngx_cache_purge-2.3.tar.gz
tar -zxvf ngx_cache_purge-2.3.tar.gz
4、下载nginx_upstream_check_module模块，该模块用于upstream健康检查
wget https://github.com/yaoweibin/nginx_upstream_check_module/archive/v0.3.0.tar.gz
tar -zxvf v0.3.0.tar.gz
5、OpenResty配置增加
cd openresty-1.13.6.1
./configure --prefix=/usr/servers --with-http_realip_module  --with-pcre  --with-luajit --add-module=./bundle/ngx_cache_purge-2.3/ --add-module=./bundle/nginx_upstream_check_module-0.3.0/ -j2 
6、编译安装
make
make install

7、OpenResty没有http模块，需要单独安装
cd /usr/servers/lualib/resty
wget https://raw.githubusercontent.com/pintsized/lua-resty-http/master/lib/resty/http_headers.lua  
wget https://raw.githubusercontent.com/pintsized/lua-resty-http/master/lib/resty/http.lua

8、项目脚本拷贝到这里
copy dynamic_eureka_balancer.lua into this dir
9、Nginx配置文件
vim /usr/servers/nginx/conf/nginx.conf
```
## Nginx配置
```

http {
	#sharing cache area
	lua_shared_dict dynamic_eureka_balancer 128m;

	init_worker_by_lua_block {
		-- init eureka balancer
		local file = require "resty.dynamic_eureka_balancer"
		local balancer = file:new({dict_name="dynamic_eureka_balancer"})
		
		--eureka server list
		balancer.set_eureka_service_url({"127.0.0.1:8888", "127.0.0.1:9999"})
		
		--The service name that needs to be monitored
		balancer.watch_service({"zuul", "client"})
	}
	
	upstream springcloud_cn {
		server 127.0.0.1:666; # Required, because empty upstream block is rejected by nginx (nginx+ can use 'zone' instead)
		
		balancer_by_lua_block {    
		
			--The zuul name that needs to be monitored
			local service_name = "zuul"
			
			local file = require "resty.dynamic_eureka_balancer"
			local balancer = file:new({dict_name="dynamic_eureka_balancer"}) 
			
			--balancer.ip_hash(service_name) --IP Hash LB
			balancer.round_robin(service_name) --Round Robin LB
		}
	}

    server {
        listen       80;
        server_name  localhost;
		
		location / {
			proxy_pass  http://springcloud_cn/;
			proxy_set_header Host $http_host;
			proxy_set_header X-Real-IP $remote_addr;
			proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
			proxy_set_header X-Forwarded-Proto  $scheme;
		}

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
	}
}
```
