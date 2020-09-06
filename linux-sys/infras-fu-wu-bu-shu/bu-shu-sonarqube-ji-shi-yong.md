# 部署 sonarqube 及使用

### Prerequisite

* **java Oracle JRE 11 or OpenJDK 11**

```text
# download jre11 rpm package
# install jre11 rpm
rpm -i jdk-11.0.5_linux-x64_bin.rpm
```

* **Linux installation prerequisite**

```text
# If you're running on Linux, you must ensure that:

* vm.max_map_count is greater or equals to 262144
* fs.file-max is greater or equals to 65536
* the user running SonarQube can open at least 65536 file descriptors
* the user running SonarQube can open at least 4096 threads

# You can see the values with the following commands:
sysctl vm.max_map_count
sysctl fs.file-max
ulimit -n
ulimit -u

# You can set them dynamically for the current session by running the following commands as root:
sysctl -w vm.max_map_count=262144
sysctl -w fs.file-max=65536
ulimit -n 65536
ulimit -u 4096

# To set these values more permanently, you must update either /etc/sysctl.d/99-sonarqube.conf (or /etc/sysctl.conf as you wish) to reflect these values.
```

* **postgresql &gt;= v10 centos7 为例（postgreSQL 11, Centos7, x86\_64）**

```text
# Install the repository RPM
yum install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm

# Install the client packages:
yum install postgresql11

# Optionally install the server packages:
yum install postgresql11-server

# Optionally initialize the database and enable automatic start:

/usr/pgsql-11/bin/postgresql-11-setup initdb
systemctl enable postgresql-11\
systemctl start postgresql-11

# PostgreSQL会自动创建postgres用户, 创建数据库之前, 要用postgres用户或root用户登录并重置postgres用户密码.
su -l postgres

# 连接数据库, psql命令会激活PostgreSQL数据库终端:
psql

# 会输出下列说明连接进入了PostgreSQL数据库:
-bash-4.2$ psql
psql (11.5)
Type "help" for help.
postgres=# help
You are using psql, the command-line interface to PostgreSQL.
Type:  \copyright for distribution terms
       \h for help with SQL commands
       \? for help with psql commands
       \g or terminate with semicolon to execute query
       \q to quit

# 重置postgres用户密码:
postgres=# \password

# 开启远程访问
    vim /var/lib/pgsql/11/data/postgresql.conf
    修改#listen_addresses = 'localhost'  为  listen_addresses=‘*'

# 信任远程连接
    vim /var/lib/pgsql/11/data/pg_hba.conf
    修改如下内容，信任指定服务器连接

    # IPv4 local connections:
    host    all            all      127.0.0.1/32      trust
    host    all            all      10.xx.xx.6/32（需要连接的服务器IP）  trust

# 重启数据库生效
systemctl restart postgresql-11

# 创建新用户
CREATE USER sonarqube  WITH PASSWORD 'Wood!nHo13';

# 创建数据库
CREATE SCHEMA sonar_schema;
ALTER USER sonarqube SET search_path to sonarqube;
CREATE DATABASE sonarqube OWNER sonarqube;
GRANT ALL PRIVILEGES ON DATABASE sonarqube to sonarqube;
DROP DATABASE sonarqube;
```

_PostgreSQL 设置_  
if you want to use a custom schema and not the default "public" one, the PostgreSQL search\_path property must be set:

```text
ALTER USER sonarqube SET search_path to sonarqube;
```

### Installing the Web Server

* **download from here**

SonarQube cannot be run as root on Unix-based systems, so create a dedicated user account to use for SonarQube if necessary.

```text
useradd sonar
```

* **Setting the Access to the Database**

$SONARQUBE-HOME \(below\) refers to the path to the directory where the SonarQube distribution has been unzipped.

Edit `$SONARQUBE-HOME/conf/sonar.properties` to configure the database settings.

Templates are available for every supported database. Just uncomment and configure the template you need and comment out the lines dedicated to H2:

```text
Example for PostgreSQL

sonar.jdbc.username=sonarqube
sonar.jdbc.password=mypassword
sonar.jdbc.url=jdbc:postgresql://localhost/sonarqube
```

* **Adding the JDBC Driver**

Drivers for the supported databases \(except Oracle\) are already provided. Do not replace the provided drivers; they are the only ones supported.

* **Configuring the Elasticsearch storage path**

By default, Elasticsearch data is stored in `$SONARQUBE-HOME/data`, but this is not recommended for production instances. Instead, you should store this data elsewhere, ideally in a dedicated volume with fast I/O. Beyond maintaining acceptable performance, doing so will also ease the upgrade of SonarQube.

Edit`$SONARQUBE-HOME/conf/sonar.properties` to configure the following settings:

```text
sonar.path.data=/var/sonarqube/data
sonar.path.temp=/var/sonarqube/temp
```

The user used to launch SonarQube must have read and write access to those directories.

```text
chown sonarqube:sonarqube /var/sonarqube/data
chown sonarqube:sonarqube /var/sonarqube/temp
```

* **SonarQube中文界面**

> 下载中文语言包 sonar-l10n-zh-plugin.jar  
> 然后，将其放入sonar安装目录的 extensions/plugins 目录下  
> [https://search.maven.org/search?q=sonar-l10n-zh-plugin](https://links.jianshu.com/go?to=https%3A%2F%2Fsearch.maven.org%2Fsearch%3Fq%3Dsonar-l10n-zh-plugin)

### Starting the Web Server

The default port is "9000" and the context path is "/". These values can be changed in $SONARQUBE-HOME/conf/sonar.properties:

```text
sonar.web.host=xxx.xxx.xxx.xxx
sonar.web.port=80
sonar.web.context=/sonarqube
```

Execute the following script to start the server:

```text
bin//sonar.sh start
```

You can now browse SonarQube at [http://localhost:9000](https://links.jianshu.com/go?to=http%3A%2F%2Flocalhost%3A9000) \(the default System administrator credentials are admin/admin\).0人点赞[CICD](https://www.jianshu.com/nb/38814403)  


