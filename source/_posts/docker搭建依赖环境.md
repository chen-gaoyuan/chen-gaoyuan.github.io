---
title: docker搭建依赖环境
---

## 安装 docker

```sh
# step 1: 安装必要的一些系统工具
sudo apt-get update
sudo apt-get -y install apt-transport-https ca-certificates curl software-properties-common

# step 2: 安装 GPG 证书
curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -

# Step 3: 写入软件源信息
sudo add-apt-repository "deb [arch=amd64] https://mirrors.aliyun.com/docker-ce/linux/ubuntu \$(lsb_release -cs) stable"

# Step 4: 更新并安装 Docker-CE
sudo apt-get -y update
sudo apt-get -y install docker-ce

# Step 5: 配置加速器
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": [
      "https://5g4dpu5e.mirror.aliyuncs.com",
      "https://docker.mirrors.ustc.edu.cn",
      "http://f1361db2.m.daocloud.io",
      "https://registry.docker-cn.com"
    ]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

## 创建 network

创建一个公用的 network,使得加入此 network 的容器能互相访问

```sh
sudo docker network create --driver bridge common-network
```

## 单点部署 mysql

安装 mariadb, 使用 utf8mb4 字符集.

```sh
sudo docker run \
    --name mariadb \
    --network common-network \
    -e MYSQL_ROOT_PASSWORD={{MYSQL密码}} \
    -p 3306:3306 \
    -v $PWD/mysql/conf.d:/etc/mysql/conf.d \
    -v $PWD/mysql/data:/var/lib/mysql \
    -d mariadb \
    --character-set-server=utf8mb4 \
    --collation-server=utf8mb4_unicode_ci
```

连接数据库

```sh
sudo docker exec -it mariadb mysql -hlocalhost -uroot -p{{MYSQL密码}}
```

## 单点部署 nacos

用 nacos 来做集群中的 注册中心 与 配置中心.

```sh
sudo docker run \
    --name nacos-server \
    --network common-network \
    -e MODE=standalone \
    -e PREFER_HOST_MODE=hostname \
    -e NACOS_SERVER_PORT=8848 \
    -p 8848:8848 \
    -d nacos/nacos-server
```

## 单点部署 nacos-dns

用 nacos-dns 做 nacos 的 dns 服务发现,
使用 SRV 请求可以获得服务的 IP,端口,权值,优先级

```sh
sudo docker run \
    --name nacos-dns \
    --network common-network \
    -e "NODE_ENV=production" \
    -e "LOG_LEVEL=WARN" \
    -e "NACOS_LIST=nacos-server:8848" \
    -e "NACOS_NAMESPACEID=public" \
    -e "DOM_UPDATE_INTERVAL=5000" \
    -e "CACHE_TTL=5" \
    -p 15353:15353/tcp \
    -p 15353:15353/udp \
    -v $PWD/nacos-dns/logs:/home/nacos-dns/logs \
    -d quickts/nacos-dns
```

## 单点部署 redis

主要用来存储临时数据, 如今天签到数据, 本周大奖赛数据等
使用 aof 方式持久化

```sh
sudo docker run \
    --name redis \
    --network common-network \
    -p 6379:6379 \
    -v $PWD/redis/data:/data \
    -d redis redis-server \
    --appendonly yes \
    --requirepass {{REDIS密码}}
```

## 单点部署 rabbitmq

使用 rabbitmq 做事件中心

```sh
sudo docker run \
    --name rabbitmq \
    --network common-network \
    -e RABBITMQ_DEFAULT_USER=root \
    -e RABBITMQ_DEFAULT_PASS={{RABBITMQ密码}} \
    -p 15672:15672 \
    -p 5672:5672 \
    -d rabbitmq:management
```

## 单点部署 kong

dns 请求是用 udp 发送的, 而 docker 中使用 udp 有些问题, 所以这里使用 host 的网络类型.
因为修改了 dns, 所以只能配置 postgres 的 ip.

```sh
sudo docker run \
    --name kong \
    --network host \
    -e "KONG_DATABASE=off" \
    -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" \
    -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" \
    -e "KONG_PROXY_ERROR_LOG=/dev/stderr" \
    -e "KONG_ADMIN_ERROR_LOG=/dev/stderr" \
    -e "KONG_PROXY_LISTEN=0.0.0.0:8000, 0.0.0.0:8443 ssl" \
    -e "KONG_ADMIN_LISTEN=0.0.0.0:8001, 0.0.0.0:8444 ssl" \
    -e "KONG_DNS_RESOLVER=127.0.0.1:15353" \
    -d kong
```

如果想使用 80 443 接口, 可以将 80 443 映射到 8000 8443, 重启会失效.

```sh
sudo iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 8000
sudo iptables -t nat -A PREROUTING -p tcp --dport 443 -j REDIRECT --to-port 8443
```

安装 konga, 网页上配置 kong 的地址时, 需配置http://172.17.0.1:8001

```sh
sudo docker run \
    --name konga \
    --network common-network \
    -e "TOKEN_SECRET=quickts" \
    -e "NODE_ENV=production" \
    -p 1337:1337 \
    -d pantsel/konga
```

### 运行程序

mysql 中创建数据库 quickts, 使用 sql/quickts.sql 初始化
拷贝.env_example 到.env, 修改对应参数
vscode 中可以直接 f5 调试运行

### 端口介绍

| 端口  | 类型      | 归属           | 作用               |
| ----- | --------- | -------------- | ------------------ |
| 3306  | tcp       | mysql(mariadb) | 数据库连接         |
| 8848  | http      | nacos          | 服务注册与服务发现 |
| 15353 | tcp & udp | nacos-dns      | dns 服务发现       |
| 6379  | tcp       | redis          | 数据库连接         |
| 5672  | tcp       | rabbitmq       | 消息队列连接       |
| 15672 | http      | rabbitmq       | web 控制台         |
| 8000  | http      | kong           | API 网关入口       |
| 8443  | https     | kong           | API 网关入口       |
| 8001  | http      | kong           | Open-API           |
| 8444  | https     | kong           | Open-API           |
| 1337  | http      | konga          | web 控制台         |
