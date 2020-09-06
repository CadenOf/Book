# pypi server 搭建

#### 通过 pip 安装 pip server

```text
$ pip install pypiserver
```

#### 安装 周边服务

```text
$ pip install passlib pypiserver gunicorn
```

#### 设置密码

```text
$ yum install httpd-tools

# 生成密码文件
$ htpasswd -c /root/.pypipasswd admin
password:
password agin:
```

#### 使用 systemd 管理 pypiserver

```text
$ tee /usr/lib/systemd/system/pypi.service <<-'EOF'
[Unit]
Description=pypi-server 
daemonAfter=network.target

# -P 使用密码文件，-a update,download 更新，下载都需要密码验证
[Service]
ExecStart=/usr/local/bin/pypi-server \
          -p 8080 \    
          -P /root/.pypipasswd -a update,download \
          -r /var/www/pypi \    
          --log-file /var/log/pypi/pypi.log
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s TERM $MAINPID

[Install]
WantedBy=multi-user.target
EOF

# 如果 gunicorn 不在 /usr/local/bin/gunicorn 路径下，则可软链过去
$ pwd
/root/.pyenv/shims
$ ln -s /root/.pyenv/shims/gunicorn /usr/local/bin/gunicorn

# 创建 /var/www/pypi 路径
$ mkdir -p /var/www/pypi
```

#### 启动 pypiserver 

```text
$ systemctl daemon-reload
$ systemctl start pypi
$ systemctl status pypi
● pypi.service - pypi-server daemon
   Loaded: loaded (/etc/systemd/system/pypi.service; disabled; vendor preset: disabled)
   Active: active (running) since 二 2020-03-03 16:15:32 CST; 47min ago
  Process: 8147 ExecStop=/bin/kill -s TERM $MAINPID (code=exited, status=0/SUCCESS)
 Main PID: 8149 (pypi-server)
    Tasks: 1
   Memory: 17.5M
   CGroup: /system.slice/pypi.service
           └─8149 /root/.pyenv/versions/3.7.2/bin/python3.7 /root/.pyenv/versions/3.7.2/bin/pypi-server -p 8080 -P /root/.pypipasswd -a update,download -r /var/www/pypi --log-file /var/log/pypi/pypi.log


3月 03 16:15:32 pro-pypi-server-200303 systemd[1]: Started pypi-server daemon.

```

#### 验证 pypiserver 功能

```text
$ cd /var/www/pypi

$ pip download pypiserver==1.3.2 --trusted-host mirrors.cloud.aliyuncs.com
Looking in indexes: http://mirrors.cloud.aliyuncs.com/pypi/simple/
Collecting pypiserver==1.3.2
  Downloading http://mirrors.cloud.aliyuncs.com/pypi/packages/80/b1/76541cbc2bfea31e3429d9b94ea935de438d1e35bca6a8047195a9d4b2be/pypiserver-1.3.2-py2.py3-none-any.whl (75 kB)
     |████████████████████████████████| 75 kB 12.5 MB/s
  Saved ./pypiserver-1.3.2-py2.py3-none-any.whl
Successfully downloaded pypiserver

$ pip search -i http://172.19.60.23:8080 pypiserver
pypiserver (1.3.2)  - 1.3.2
  INSTALLED: 1.3.2 (latest)

$ pip search -i http://172.19.60.23:8080 pypiserver
pypiserver (1.3.2)  - 1.3.2
  INSTALLED: 1.3.2 (latest)
```

#### 客户端配置

```text
# 加上自己的内部源，配置多个 pypi 源
$ tee ~/.pip/pip.conf <<-'EOF'
[global]
index-url=https://172.19.60.23:8080/simple/
extra-index-url=
  https://pypi.xxxx.com/simple/
  http://mirrors.cloud.aliyuncs.com/pypi/simple/

[install]
trusted-host=
  mirrors.cloud.aliyuncs.com
  172.19.60.23:8080
  pypi.graviti.cn:8080
EOF


$ tee  ~/.pypirc <<-'EOF'
[distutils]
index-servers =
  internal

[internal]
repository: http://172.19.60.23:8080
username: admin
password: xxxx
EOF
```

#### 使用 pypi 源

```text
## 搜索包
$ pip search -i http://172.28.70.126:8080 $package_name

## 安装内部源的包
$ pip install -i http://172.28.70.126:8080 $package_name

## 上传包
$ pip3 install twine
$ twine upload -r internal xxx.tar.gz

## 访问 web 查看 package list
http://pypi.xxxx.com/simple

```

