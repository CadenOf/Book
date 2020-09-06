# Kong Api Gateway

## Why Kong?

Kong 可以充当微服务请求的网关（或 sidecar），同时通过插件提供负载平衡，日志记录，身份验证，速率限制，转换等功能。

![](../.gitbook/assets/image.png)

**高性能**：亚毫秒级的处理延迟，可支持关键任务用例和高吞吐量；

**可扩展性**：具有可插拔的架构体系，可通过 Plugin SDK 扩展 Lua 或 GoLang 中的 Kong；

**可移植性**：可在任意平台，任意云上运行，并能通过当前的 Ingress Controller 天然支持 Kubernetes。



## Kong 的特性

* 云原生\(**Cloud-Native**\)：Kong 可以在 Kubernetes 或物理环境上运行；
* 动态**负载平衡**\(**Dynamic Load Balancing**\)：跨多个上游服务的负载平衡业务。
* 基于哈希的负载平衡\(**Hash-based Load Balancing**\)：一致的散列/粘性会话的负载平衡。
* 断路器\(**Circuit-Breaker**\)：智能跟踪不健康的上游服务。
* 健康检查\(**Health Checks**\)：主动和被动监控您的上游服务。
* **服务发现**\(**Service Discovery**\)：解决如 Consul 等第三方 DNS 解析器的 SRV 记录。
* 无服务器\(**Serverless**\)：从 Kong 中直接调用和保证 AWS 或 OpenWhisk 函数安全。
* **WebSockets**：通过 **WebSockets** 与上游服务进行通信。
* OAuth2.0：轻松的向 API 中添加 OAuth2.0 认证。
* **日志记录**\(**Logging**\)：通过 HTTP、TCP、UDP 记录请求或者相应的日志，存储在磁盘中。
* **安全**\(**Security**\)：ACL，Bot 检测，IPs 白名单/黑名单等。
* 系统日志\(**Syslog**\)：记录信息到系统日志。
* SSL：为基础服务或 API 设置特定的 SSL 证书。
* **监视**\(**Monitoring\)**：能够实时对关键负载和性能指标进行监控。
* 转发代理\(**Forward Proxy**\)：使端口连接到中间透明的 HTTP 代理。
* **认证**\(**Authentications**\)：支持 HMAC，JWT 和 BASIC 方式进行认证等等。
* **速率限制**\(**Rate-limiting**\)：基于多个变量的阻塞和节流请求。
* 转换\(**Transformations**\)：添加、删除或操作 HTTP 请求和响应。
* **缓存**\(**Caching**\)：在代理层进行缓存和服务响应。
* 命令行工具\(CLI\)：能够通过命令行控制 Kong 的集群。
* REST API：可以通过 REST API 灵活的操作 Kong。
* GEO复制：在不同的区域，配置总是最新的。
* **故障检测与恢**复\(**Failure Detection & Recovery**\)：如果 Cassandra 节点失效，Kong 并不会受影响。
* 群集\(**Clustering**\)：所有的 Kong 节点会自动加入群集，并更新各个节点上的配置。
* 可扩展性\(**Scalability**\)：通过添加节点，实现水平缩放。
* 性能\(**Performance**\)：通过缩放和使用 Nigix，Kong 能够轻松处理负载。
* 插件\(**Plugins**\)：基于插件的可扩展体系结构，能够方便的向 Kong 和 API 添加功能。

## 使用 docker 方式部署 kong

#### 部署 kong-database postgres

```text
$ docker run -d --name kong-database \
                -p 5432:5432 \
                -e "POSTGRES_USER=kong" \
                -e "POSTGRES_DB=kong" \
                -e "POSTGRES_PASSWORD=passwd123" \
                postgres:9.6
```

#### 初始化数据库

```text
$ docker run --rm \
    --link kong-database:kong-database \
    -e "KONG_DATABASE=postgres" \
    -e "KONG_PG_HOST=kong-database" \
    -e "KONG_PG_PASSWORD=passwd123" \
    kong kong migrations bootstrap
```

#### 启动 kong

```text
$ docker run -d --name kong-server \
    --link kong-database:kong-database \
    -e "KONG_DATABASE=postgres" \
    -e "KONG_PG_HOST=kong-database" \
    -e "KONG_PG_PASSWORD=passwd123" \
    -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" \
    -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" \
    -e "KONG_PROXY_ERROR_LOG=/dev/stderr" \
    -e "KONG_ADMIN_ERROR_LOG=/dev/stderr" \
    -e "KONG_ADMIN_LISTEN=0.0.0.0:8001, 0.0.0.0:8444 ssl" \
    -p 8000:8000 \
    -p 8443:8443 \
    -p 8001:8001 \
    -p 8444:8444 \
    kong
```

#### 自定义配置启动服务

启动 kong 时，可通过传入前缀为 `KONG_` 的环境变量来服务配置文件中的任意配置，例如

```text
$ docker run -d --name kong \
    -e "KONG_DATABASE=postgres"
    -e "KONG_PG_HOST=kong-database" \
    -e "KONG_LOG_LEVEL=info" \
    -e "KONG_CUSTOM_PLUGINS=helloworld" \
    -e "KONG_PG_HOST=1.1.1.1" \
    -e "KONG_ADMIN_LISTEN=0.0.0.0:8001, 0.0.0.0:8444 ssl" \
    -p 8000:8000 \
    -p 8443:8443 \
    -p 8001:8001 \
    -p 8444:8444 \
    kong
```

#### Reload 配置

在 kong 服务还在运行时，若有配置修改，则可通过 `reload` 命令更新服务

```text
$ docker exec -it kong kong reload
```



## 使用 konga dashboard

初始化 konga 数据库 （使用 kong server 的 postgres 数据库）

```text
$ docker run --link kong-database:kong-database \
             --rm  pantsel/konga:latest \
             -c prepare \
             -a postgres \
             -u postgres://kong:passwd123@kong-database:5432/konga
```

```text

$ docker run -p 1337:1337 
          --network {{kong-network}} \ // optional
          -e "TOKEN_SECRET={{somerandomstring}}" \
          -e "DB_ADAPTER=the-name-of-the-adapter" \ // 'mongo','postgres','sqlserver'  or 'mysql'
          -e "DB_HOST=your-db-hostname" \
          -e "DB_PORT=your-db-port" \ // Defaults to the default db port
          -e "DB_USER=your-db-user" \ // Omit if not relevant
          -e "DB_PASSWORD=your-db-password" \ // Omit if not relevant
          -e "DB_DATABASE=your-db-name" \ // Defaults to 'konga_database'
          -e "DB_PG_SCHEMA=my-schema"\ // Optionally define a schema when integrating with prostgres
          -e "NODE_ENV=production" \ // or 'development' | defaults to 'development'
          --name konga \
          pantsel/konga


# customize      

$ docker run -d --name konga-client \
--link kong-database:kong-database \
-p 1337:1337 \
-e "DB_ADAPTER=postgres" \
-e "DB_HOST=kong-database" \
-e "DB_PORT=5432" \
-e "DB_USER=kong" \
-e "DB_PASSWORD=passwd123" \
-e "DB_DATABASE=konga" \
-e "NODE_ENV=production" \
pantsel/konga
```

安装完成，通过 http://{your\_ip}:1337 即可访问，注册用户后登录（下图 dashboard 默认为空，成功添加 kong 服务后才会显示）

![konga dashboard](../.gitbook/assets/image%20%2810%29.png)

添加一个 kong-server connection

![](../.gitbook/assets/image%20%2811%29.png)

![add connection](../.gitbook/assets/image%20%286%29.png)

![Active Connection](../.gitbook/assets/image%20%282%29.png)

![](../.gitbook/assets/image%20%283%29.png)

添加 service（添加 baidu.com 为例测试）

![add service](../.gitbook/assets/image%20%289%29.png)

在新加的 service 下添加 route

![add route](../.gitbook/assets/image%20%285%29.png)

![](../.gitbook/assets/image%20%284%29.png)

{% hint style="danger" %}
需要特别注意的是，在 Hosts, Paths, Methus 填写时要“回车“一下，输入的内容才会添加上去，不然会出现上图中的红色报错
{% endhint %}

测试访问成功

![](../.gitbook/assets/image%20%288%29.png)

