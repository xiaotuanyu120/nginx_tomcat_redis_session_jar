---
title: redis+tomcat+nginx实现session共享
date: 2016-09-27 12:59:00
categories: linux
tags: [redis,session,tomcat,nginx]
---
### 环境介绍
**软件版本**
- nginx: 1.10.1
- tomcat: 6.0.45
- redis: 3.2.4

**软件架构**
| 软件名称 | HOST |
| :---: | :---: |
| nginx | 192.168.110.4|
| tomcat01 | 192.168.110.5|
| tomcat02 | 192.168.110.6|
| redis | 192.168.110.5|

<!--more-->

### 安装并配置nginx
在nginx节点执行以下命令
**安装nginx**
``` bash
yum install epel-release -y
yum install nginx -y
chkconfig nginx on
```
**配置nginx**
``` bash
# 新建nginx虚拟机配置文件
vim /etc/nginx/conf.d/sstest.conf
******************************
upstream ss {
    server 192.168.110.5:8080;
    server 192.168.110.6:8080;
}

server {
    listen 80;
    server_name 192.168.110.4 default_server;
    location / {
        proxy_pass http://ss;
    }
}
******************************
service nginx restart
```

### 安装并配置tomcat
在两台tomcat节点执行以下命令

**安装tomcat**
``` bash
# 下载并安装jdk1.6
# 需要注册oracle帐号才能在下面下载
http://www.oracle.com/technetwork/java/javase/downloads/java-archive-downloads-javase6-419409.html
chmod u+x jre-6u45-linux-x64.bin
mv jre1.6.0_45 /usr/local/
ln -s /usr/local/jre1.6.0_45/ /usr/local/jdk
vi /etc/profile.d/java-env.sh
*******************************
JAVA_HOME=/usr/local/jdk
JRE_HOME=${JAVA_HOME}/jre
PATH=$PATH:${JAVA_HOME}/bin:${JRE_HOME}/bin
CLASSPATH=${JAVA_HOME}/lib:${JRE_HOME}/lib
*******************************
source /etc/profile.d/java-env.sh

# 安装tomcat
wget http://mirror.rise.ph/apache/tomcat/tomcat-6/v6.0.45/bin/apache-tomcat-6.0.45.tar.gz
tar zxf apache-tomcat-6.0.45.tar.gz
mv apache-tomcat-6.0.45 /usr/local/tomcat
```


**配置tomcat**
``` bash
# 配置tomcat虚拟主机
vim /usr/local/tomcat/conf/server.xml
************************
<Host name="localhost"  appBase=""
      unpackWARs="true" autoDeploy="true">
  <Context path="" docBase="/data/webapps" reloadable="true" />
  ......
</Host>
************************

vim /usr/local/tomcat/conf/context.xml
******************************
    <Valve className="com.radiadesign.catalina.session.RedisSessionHandlerValve"/>
    <Manager className="com.radiadesign.catalina.session.RedisSessionManager" 
     host="47.90.80.156"
     port="6379"
     database="0"
     maxInactiveInterval="60"
     />
******************************
```
Valve要在Manager之前
注意className，网上的都不一样，需要按照自己下载jar包的版本进行调整

**编写测试程序**
``` bash
# 创建web程序目录
mkdir -p /data/webapps/{WEB-INF,META-INF,classes,lib}
cd /data/webapps

# 编写首页程序
# ss02上把Tomcat01更换成Tomcat02
vim index.jsp
***********************
<%@ page language="java" %>
<html>
  <head><title>Tomcat01</title></head>
  <body>
    <h1><font color="red">Tomcat01.magedu.com</font></h1>
    <table align="centre" border="1">
      <tr>
        <td>Session ID</td>
    <% session.setAttribute("magedu.com","magedu.com"); %>
        <td><%= session.getId() %></td>
      </tr>
      <tr>
        <td>Created on</td>
        <td><%= session.getCreationTime() %></td>
     </tr>
    </table>
  </body>
</html>
***********************
```


### 安装并配置redis
**安装redis**
``` bash
wget http://download.redis.io/releases/redis-2.8.24.tar.gz
tar zxf redis-2.8.24.tar.gz 
cd redis-2.8.24
make
make install
```

**拷贝redis启动脚本**
``` bash
cp utils/redis_init_script /etc/init.d/redis
vi /etc/init.d/redis
******************************
# 在第2行添加
#chkconfig: 2345 80 90 

# 修改配置文件位置
CONF="/etc/redis/${REDISPORT}.conf"
******************************
chkconfig redis on
```

**配置redis**
``` bash
mkdir /etc/redis
cp redis.conf /etc/redis/6379.conf
vi /etc/redis/6379.conf
******************************
bind 0.0.0.0
daemonize yes
******************************
```

**拷贝jar包到tomcat**
``` bash
wget https://raw.githubusercontent.com/xiaotuanyu120/nginx_tomcat6_redis2.8/master/tomcat-redis-session-manager-1.2-tomcat-6.jar
wget https://raw.githubusercontent.com/xiaotuanyu120/nginx_tomcat6_redis2.8/master/jedis-2.1.0.jar
wget https://raw.githubusercontent.com/xiaotuanyu120/nginx_tomcat6_redis2.8/master/commons-pool-1.6.jar

mv tomcat-redis-session-manager-1.2-tomcat-6.jar jedis-2.1.0.jar commons-pool-1.6.jar /usr/local/tomcat/lib/
```
根据jdk和tomcat版本,需要的jar包完全不一样,不同的jar包版本对应的tomcat Context配置亦不同

**启动redis、重启tomcat服务**
``` bash
service redis start

# 在两个tomcat节点上重启tomcat
/usr/local/tomcat/bin/catalina.sh stop
/usr/local/tomcat/bin/catalina.sh start
```

### 效果测试
``` bash
# 分别访问tomca01和tomcat02
curl -I http://47.90.80.156:8080/
HTTP/1.1 200 OK
Server: Apache-Coyote/1.1
Set-Cookie: JSESSIONID=7E548773A2181763F25C709490677B41; Path=/
Content-Type: text/html
Content-Length: 370
Date: Tue, 27 Sep 2016 08:32:06 GMT

curl -I http://47.90.80.167:8080/
HTTP/1.1 200 OK
Server: Apache-Coyote/1.1
Set-Cookie: JSESSIONID=2E10147173A1565E1BAA79D766A29410; Path=/
Content-Type: text/html
Content-Length: 370
Date: Tue, 27 Sep 2016 08:32:29 GMT

# 负载均衡端访问
web端访问，session值统一
```