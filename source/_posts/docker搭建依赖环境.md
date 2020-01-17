---
title: docker搭建依赖环境
---

## 创建 network

创建一个公用的 network,使得加入此 network 的容器能互相访问

```sh
sudo docker network create --driver bridge common-network
```

## 单点部署 mysql

安装 mariadb, 使用 utf8mb4 字符集.
如果机器够多, 可以考虑主从部署.

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
如果机器够多可以多节点部署 nacos.

创建数据库 nacos_config, 使用 sql/nacos.sql 初始化数据库.
如果想自定义密码, 使用[BCrypt Rounds 12](https://www.jisuan.mobi/p163u3BN66Hm6JWx.html)计算出密文密码, 并替换表 nacos_config.users 中的值.

```sh
sudo docker run \
    --name nacos-server \
    --network common-network \
    -e PREFER_HOST_MODE=hostname \
    -e MODE=standalone \
    -e SPRING_DATASOURCE_PLATFORM=mysql \
    -e MYSQL_MASTER_SERVICE_HOST=mariadb \
    -e MYSQL_MASTER_SERVICE_PORT=3306 \
    -e MYSQL_MASTER_SERVICE_USER=root \
    -e MYSQL_MASTER_SERVICE_PASSWORD={{MYSQL密码}} \
    -e MYSQL_MASTER_SERVICE_DB_NAME=nacos_config \
    -e MYSQL_SLAVE_SERVICE_HOST=mariadb \
    -e MYSQL_SLAVE_SERVICE_PORT=3306 \
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

先安装 postgres, kong 与 konga 需要使用 postgres.
只能使用 9.6 的数据库.

```sh
sudo docker run \
    --name postgres \
    --network common-network \
    -e "POSTGRES_USER=kong" \
    -e "POSTGRES_DB=kong" \
    -p 5432:5432 \
    -d postgres:9.6
```

初始化 kong 的数据库

```sh
sudo docker run --rm \
    --network common-network \
    -e "KONG_DATABASE=postgres" \
    -e "KONG_PG_HOST=postgres" \
    kong kong migrations bootstrap
```

dns 请求是用 udp 发送的, 而 docker 中使用 udp 有些问题, 所以这里使用 host 的网络类型.
因为修改了 dns, 所以只能配置 postgres 的 ip.

```sh
sudo docker run \
    --name kong \
    --network host \
    -e "KONG_DATABASE=postgres" \
    -e "KONG_PG_HOST=127.0.0.1" \
    -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" \
    -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" \
    -e "KONG_PROXY_ERROR_LOG=/dev/stderr" \
    -e "KONG_ADMIN_ERROR_LOG=/dev/stderr" \
    -e "KONG_PROXY_LISTEN=0.0.0.0:8000, 0.0.0.0:8443 ssl" \
    -e "KONG_ADMIN_LISTEN=0.0.0.0:8001, 0.0.0.0:8444 ssl" \
    -e "KONG_DNS_RESOLVER=127.0.0.1:15353" \
    -v $PWD/kong/logs:/usr/local/kong/logs \
    -v $PWD/kong/conf:/etc/kong \
    -d kong
```

如果想使用 80 443 接口, 可以将 80 443 映射到 8000 8443, 重启会失效.

```sh
sudo iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 8000
sudo iptables -t nat -A PREROUTING -p tcp --dport 443 -j REDIRECT --to-port 8443
```

初始化 konga 的数据库

```sh
sudo docker run \
    --network common-network \
    --rm pantsel/konga -c prepare -a postgres -u postgresql://kong:@postgres:5432/konga
```

安装 konga, 网页上配置 kong 的地址时, 需配置http://172.17.0.1:8001

```sh
sudo docker run \
    --name konga \
    --network common-network \
    -e "TOKEN_SECRET=quickts" \
    -e "DB_ADAPTER=postgres" \
    -e "DB_URI=postgresql://kong:@postgres:5432/konga" \
    -e "NODE_ENV=production" \
    -p 1337:1337 \
    -d pantsel/konga
```

### 运行程序

mysql 中创建数据库 quickts, 使用 sql/quickts.sql 初始化
拷贝.env_example 到.env, 修改对应参数
vscode 中可以直接 f5 调试运行
