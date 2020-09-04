# Kong Api Gateway

## Why Kong?

Kong 可以充当微服务请求的网关（或 sidecar），同时通过插件提供负载平衡，日志记录，身份验证，速率限制，转换等功能。

![](../.gitbook/assets/image.png)

**高性能**：亚毫秒级的处理延迟，可支持关键任务用例和高吞吐量；

**可扩展性**：具有可插拔的架构体系，可通过 Plugin SDK 扩展 Lua 或 GoLang 中的 Kong；

**可移植性**：可在任意平台，任意云上运行，并能通过当前的 Ingress Controller 天然支持 Kubernetes。



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















