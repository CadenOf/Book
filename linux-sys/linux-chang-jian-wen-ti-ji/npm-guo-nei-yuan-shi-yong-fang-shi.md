# npm 国内源使用方式

#### 直接配置镜像库方式

```text
# 默认为官方
$ npm config get registry
https://registry.npmjs.org/"

# 设置为淘宝镜像
$ npm config set registry https://registry.npm.taobao.org
$ npm config get registry
https://registry.npm.taobao.org/
```

#### 使用 cnpm

安装淘宝的cnpm，然后在使用时直接将npm命令替换成cnpm命令即可

```text
$ npm install -g cnpm --registry=https://registry.npm.taobao.org/
```

#### 使用 http 方式

关闭 npm 的 https，再将 registry 设置为 http 地址

```text
$ npm config set strict-ssl false
$ npm config set registry "http://registry.npmjs.org/"
```

#### 自建代理方式

 在国外自建 vps + http 代理

```text
npm config set proxy=http://代理服务器ip:代理服务器端口
```

