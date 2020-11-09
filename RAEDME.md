# docker keepalived nginx集群 

## 用dockerfile，构建容器

### 准备脚本文件

entrypoint.sh  一定要在linux里面编写这个文件

这里是用来启动keepalived 和 nginx 用的，nginx -g "daemon off;"的意思是运行完了也不退出

```sh
#!/bin/sh
#/usr/sbin/keepalived -n -l -D -f /etc/keepalived/keepalived.conf --dont-fork --log-console &
/usr/sbin/keepalived -D -f /etc/keepalived/keepalived.conf
nginx -g "daemon off;"
```

### 写dockerfile

Dockerfile

这里用了nginx镜像，然后在里面安装了keepalived，然后运行两个程序

要注意的是，运行的时候看看 nginx和keepalived是否都启动了

```dockerfile
FROM nginx:1.13.5-alpine
RUN apk update && apk upgrade
RUN apk add --no-cache bash curl ipvsadm iproute2 openrc keepalived
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
CMD ["/entrypoint.sh"]
```



## 用dockercompose编排容器

### dockercompose

docker-compose.yml 

这里ip需要固定住，不然keepalived 不起作用

```yaml
version: "3.1"
services:
  nginx_master:
    build:
      context: ./
      dockerfile: ./Dockerfile
    ports:
      - 8081:80
    volumes:
      - ./index-master.html:/usr/share/nginx/html/index.html
      - ./keepalived-master.conf:/etc/keepalived/keepalived.conf
    networks:
      static-network:
        ipv4_address: 172.20.128.2
    cap_add:
      - NET_ADMIN
  nginx_slave:
    build:
      context: ./
      dockerfile: ./Dockerfile
    ports:
      - 8082:80
    volumes:
      - ./index-slave.html:/usr/share/nginx/html/index.html
      - ./keepalived-slave.conf:/etc/keepalived/keepalived.conf
    networks:
      static-network:
        ipv4_address: 172.20.128.3
    cap_add:
      - NET_ADMIN
  proxy:
    image:  haproxy:1.7-alpine
    ports:
      - 80:6301
    volumes:
      - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg
    networks:
      - static-network
networks:
  static-network:
    ipam:
      config:
        - subnet: 172.20.0.0/16
```

### keepalived-master.conf  

一定要在linux里面编写这个文件

把地址映射到 keepalived的 ip上，端口就是80，注意  interface eth0别写错

```conf
vrrp_script chk_nginx {
    script "pidof nginx"
    interval 2
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 33
    priority 200
    advert_int 1
    
    authentication {
    	auth_type PASS
    	auth_pass letmein
	}

	virtual_ipaddress {
        172.20.128.50
    }

	track_script {
        chk_nginx
    }
}

```

### keepalived-slave.conf 

一定要在linux里面编写这个文件

把地址映射到 keepalived的 ip上，端口就是80，注意  interface eth0别写错

```conf
vrrp_script chk_nginx {
    script "pidof nginx"
    interval 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 33
    priority 100
    advert_int 1
    
    authentication {
    	auth_type PASS
    	auth_pass letmein
	}

	virtual_ipaddress {
        172.20.128.50
    }

	track_script {
        chk_nginx
    }
}
```

### haproxy.cfg 

一定要在linux里面编写这个文件

bind 就是把所有的，下面的服务都绑定到，本机ip 6301端口上。

server 里面的是keepalived 映射过的地址，其实就是把keepalived映射的虚拟ip，再次映射出去

```cfg
global
	log 127.0.0.1 local0
	maxconn 4096
	daemon
	nbproc 4
	
defaults
	log 127.0.0.1 local3
	mode http
	option dontlognull
	option redispatch
	retries 2
	maxconn 2000
	balance roundrobin
	timeout connect 5000ms
	timeout client 5000ms
	timeout server 5000ms
	
frontend main
	bind *:6301
	default_backend akiwebserver
	
backend akiwebserver
    server node1 172.20.128.50:80 check inter 2000 rise 3 fall 5
```

### index-master.html 

一定要在linux里面编写这个文件

```html
<h1>master！</h1>
```

### index-slave.html 

一定要在linux里面编写这个文件

```html
<h1>slave！</h1>
```

