# 搭建 phabricator 项目/代码管理平台

### Installation Requirements

* 1 台 Linux host
* mysql &gt;= 5.5
* php &gt;= 5.2
* webserver 服务，apache
* domain name Installing Required Components

### RedHat Derivatives Install Script

```text
#!/bin/bash
confirm() {
echo "Press RETURN to continue, or ^C to cancel.";
read -e ignored 
}
RHEL_VER_FILE="/etc/redhat-release"
if [[ ! -f $RHEL_VER_FILE ]]
then
echo "It looks like you're not running a Red Hat-derived distribution."
echo "This script is intended to install Phabricator on RHEL-derived"
echo "distributions such as RHEL, Fedora, CentOS, and Scientific Linux."
echo "Proceed with caution."
confirm 
fi
echo "PHABRICATOR RED HAT DERIVATIVE INSTALLATION SCRIPT";
echo "This script will install Phabricator and all of its core dependencies.";
echo "Run it from the directory you want to install into.";
echo
RHEL_REGEX="release ([0-9]+)\."
if [[ $(cat $RHEL_VER_FILE) =~ $RHEL_REGEX ]]
then
RHEL_MAJOR_VER=${BASH_REMATCH[1]}
else
echo "Ut oh, we were unable to determine your distribution's major"
echo "version number. Please make sure you're running 6.0+ before"
echo "proceeding."
confirm 
fi
if [[ $RHEL_MAJOR_VER < 6 && $RHEL_MAJOR_VER > 0 ]]
then
echo "** WARNING **"
echo "A major version less than 6 was detected. Because of this,"
echo "several needed dependencies are not available via default repos."
echo "Specifically, RHEL 5 does not have a PEAR package for php53-*."
echo "We will attempt to install it manually, for APC. Please be careful."
confirm 
fi
echo "Phabricator will be installed to: $(pwd).";
confirm 
echo "Testing sudo/root..."
if [[ $EUID -ne 0 ]] # Check if we're root. If we are, continue.
then
sudo true
SUDO="sudo"
if [[ $? -ne 0 ]]
then
echo "ERROR: You must be able to sudo to run this script, or run it as root.";
exit 1
fi
fi
if [[ $RHEL_MAJOR_VER == 5 ]]
then
# RHEL 5's "php" package is actually 5.1. The "php53" package won't let us install php-pecl-apc.
# (it tries to pull in php 5.1 stuff) ...
yum repolist | grep -i epel 
if [ $? -ne 0 ]; then
echo "It doesn't look like you have the EPEL repo enabled. We are to add it"
echo "for you, so that we can install git."
$SUDO rpm -Uvh https://download.fedoraproject.org/pub/epel/5/i386/epel-release-5-4.noarch.rpm 
fi
YUMCOMMAND="$SUDO yum install httpd git php53 php53-cli php53-mysql php53-process php53-devel php53-gd gcc wget make pcre-devel mysql-server"
else
# RHEL 6+ defaults with php 5.3
YUMCOMMAND="$SUDO yum install httpd git php php-cli php-mysql php-process php-devel php-gd php-pecl-apc php-pecl-json php-mbstring mysql-server"
fi
echo "Dropping to yum to install dependencies..."
echo "Running: ${YUMCOMMAND}"
echo "Yum will prompt you with [Y/n] to continue installing."
$YUMCOMMAND
if [[ $? -ne 0 ]]
then
echo "The yum command failed. Please fix the errors and re-run this script."
exit 1
fi
if [[ $RHEL_MAJOR_VER == 5 ]]
then
# Now that we've ensured all the devel packages required for pecl/apc are there, let's
# set up PEAR, and install apc.
echo "Attempting to install PEAR"
wget https://pear.php.net/go-pear.phar 
$SUDO php go-pear.phar && $SUDO pecl install apc 
fi
if [[ $? -ne 0 ]]
then
echo "The apc install failed. Continuing without APC, performance may be impacted."
fi
pidof httpd 2>&1 > /dev/null 
if [[ $? -eq 0 ]]
then
echo "If php was installed above, please run: /etc/init.d/httpd graceful"
else
echo "Please remember to start the httpd with: /etc/init.d/httpd start"
fi
pidof mysqld 2>&1 > /dev/null 
if [[ $? -ne 0 ]]
then
echo "Please remember to start the mysql server: /etc/init.d/mysqld start"
fi
confirm 
if [[ ! -e libphutil ]]
then
git clone https://github.com/phacility/libphutil.git 
else
(cd libphutil && git pull --rebase)
fi
if [[ ! -e arcanist ]]
then
git clone https://github.com/phacility/arcanist.git 
else
(cd arcanist && git pull --rebase)
fi
if [[ ! -e phabricator ]]
then
git clone https://github.com/phacility/phabricator.git 
else
(cd phabricator && git pull --rebase)
fi
echo
echo
echo "Install probably worked mostly correctly. Continue with the 'Configuration Guide':";
echo
echo " https://secure.phabricator.com/book/phabricator/article/configuration_guide/";
```

### clone 必须代码仓库

```text
$ cd somewhere/ # pick some install directory
somewhere/ $ git clone https://github.com/phacility/libphutil.git
somewhere/ $ git clone https://github.com/phacility/arcanist.git
somewhere/ $ git clone https://github.com/phacility/phabricator.git
```

### Configuration

**Webserver: Configuring nginx**

```text
# /etc/nginx/nginx.conf
#user  nobody;
worker_processes  1;
#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;
#pid        logs/nginx.pid;
events {
    worker_connections  1024;
}
http {
    include       mime.types;
    default_type  application/octet-stream;
    log_format  main  '$remote_addr - $remote_user [$time_local] $http_host:$server_port $scheme "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for" '
                      '$connection $upstream_addr '
                      '$upstream_response_time $request_time $request_length';
    #access_log  logs/access.log  main;
    sendfile        on;
    #tcp_nopush     on;
    #keepalive_timeout  0;
    keepalive_timeout  65;
    #gzip  on;
    include conf.d/*.conf;
}
```

**/etc/nginx/conf.d/phab.conf**

```text
server {
  listen 80;
  server_name phabricator-qa.graviti.cn;
  root        /usr/local/phacility/phabricator/webroot;
  access_log  /var/log/phabricator/access.log main;
  error_log  /var/log/phabricator/error.log;
  client_max_body_size 64m;
  location / {
    index index.php;
    rewrite ^/(.*)$ /index.php?__path__=/$1 last;
  }
  location /index.php {
    try_files $uri =404;
    fastcgi_pass  unix:/tmp/php-cgi.sock;
    fastcgi_index   index.php;
    fastcgi_param  SCRIPT_FILENAME  /usr/local/phacility/phabricator/webroot$fastcgi_script_name;
    include fastcgi_params;
  }
}
```

### 配置环境

**添加 phd 用户**

```text
useradd phd
```

**添加软链**

```text
ln -s /usr/local/phacility/phabricator/resources/sshd/phabricator-ssh-hook.sh phabricator-ssh-hook.sh
```

**配置 phab**

直接配置 conf/local/local.json 文件

```text
{
  "security.require-https": true,
  "diffusion.ssh-port": 2224,
  "diffusion.ssh-user": "git",
  "phd.user": "phd",
  "cluster.mailers": [
    {
      "key": "local-mailer",
      "type": "sendmail",
      "options": {
        "message-id": true
      }
    }
  ],
  "repository.default-local-path": "/data/repo/",
  "storage.local-disk.path": "/data/phabricator/",
  "phabricator.base-uri": "https://phabricator.xxxx.com",
  "mysql.pass": "Wxxxxxx3",
  "mysql.user": "root",
  "mysql.host": "127.0.0.1"
}
```

或者，通过命令更改配置

```text
./bin/config set mysql.host 127.0.0.1
./bin/config set mysql.pass xxxxxx
./bin/config set mysql.port 3306
./bin/config set mysql.user root
```

更新数据库

```text
root@service-3054-190911205326 phabricator]# ./bin/storage upgrade
Before running storage upgrades, you should take down the Phabricator web
interface and stop any running Phabricator daemons (you can disable this
warning with --force).
    Are you ready to continue? [y/N] y
Loading quickstart template onto "127.0.0.1"...
Applying patch "phabricator:db.paste" to host "127.0.0.1"...
Applying patch "phabricator:20190523.myisam.01.documentfield.sql" to host "127.0.0.1"...
Applying patch "phabricator:20190718.paste.01.edge.sql" to host "127.0.0.1"...
Applying patch "phabricator:20190718.paste.02.edgedata.sql" to host "127.0.0.1"...
Applying patch "phabricator:20190718.paste.03.paste.sql" to host "127.0.0.1"...
Applying patch "phabricator:20190718.paste.04.xaction.sql" to host "127.0.0.1"...
Applying patch "phabricator:20190718.paste.05.comment.sql" to host "127.0.0.1"...
Applying patch "phabricator:20190802.email.01.storage.sql" to host "127.0.0.1"...
Applying patch "phabricator:20190802.email.02.xaction.sql" to host "127.0.0.1"...
Applying patch "phabricator:20190815.account.01.carts.php" to host "127.0.0.1"...
Applying patch "phabricator:20190815.account.02.subscriptions.php" to host "127.0.0.1"...
Applying patch "phabricator:20190816.payment.01.xaction.sql" to host "127.0.0.1"...
Applying patch "phabricator:20190816.subscription.01.xaction.sql" to host "127.0.0.1"...
Applying patch "phabricator:20190822.merchant.01.view.sql" to host "127.0.0.1"...
Applying patch "phabricator:20190909.herald.01.rebuild.php" to host "127.0.0.1"...
Storage is up to date. Use "storage status" for details.
Synchronizing static tables...
Verifying database schemata on "127.0.0.1"...

Database                Table                Name        Issues
phabricator_conpherence  conpherence_index                  Better Table Engine Available
phabricator_differential differential_revision phid         Surplus Key
phabricator_differential differential_revision key_modified Missing Key
phabricator_differential differential_revision key_phid     Missing Key
phabricator_phortune     phortune_accountemail key_account  Missing Key
phabricator_phortune     phortune_accountemail key_address  Missing Key
phabricator_phortune     phortune_accountemail key_phid     Missing Key
Applying schema adjustments...
Done.                                                                         
Completed applying all schema adjustments.
 ANALYZE  Analyzing tables...                                                 
Done.                                                                         
 ANALYZED  Analyzed 535 table(s).
```

**使用 phd 管理后台**

phd 是一个命令行程序（位于 phabricator/bin/phd ）

```text
[root@service-3054-190911205326 phabricator]# ./bin/phd help
NAME
      phd - manage daemons
SYNOPSIS
      phd command [options]
          Manage Phabricator daemons.
WORKFLOWS
。。。。。
Use help command for a detailed command reference.
Use --show-standard-options to show additional options.
```

通常情况下，你会使用:

> * phd start 启动所有后台进程;
> * phd restart 重启所有后台进程;
> * phd status 列出所有后台进程;
> * phd stop 停止所有后台进程。

如果你想要更细粒度的控制，你可以使用：

> * phd launch 启动单个后台进程;
> * phd debug 调试

### 启动 phabricator

```text
[root@service-3054-190911205326 phabricator]# ./bin/phd start
Freeing active task leases...
Freed 0 task lease(s).
Starting daemons as phd
Launching daemons:
(Logs will appear in "/var/tmp/phd/log/daemons.log".)
    (Pool: 1) PhabricatorRepositoryPullLocalDaemon
    (Pool: 1) PhabricatorTriggerDaemon
    (Pool: 1) PhabricatorFactDaemon
    (Pool: 4) PhabricatorTaskmasterDaemon
Done.
```

