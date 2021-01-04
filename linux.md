（1）打开网络适配器，启用虚拟网卡，设置virtualbox为双网卡，第一个为nat，第二个hostonly网卡
  
  
  设置虚拟机网卡的ipv4的ip为192.168.0.1 DNS为192.168.0.101

（2）首先进入修改vim，不然vim会乱跳：
sudo vi /etc/vim/vimrc.tiny
set nocompatible
set backspace=2

（3）sudo vi /etc/apt/sources.list
在网页中打开浏览器页面，复制粘贴下面的源镜像

#清华源

#默认注释了源码镜像以提高 apt update 速度，如有需要可自行取消注释

deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial main restricted universe multiverse

#deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial main restricted universe multiverse

deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-updates main restricted universe multiverse

#deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-updates main restricted universe multiverse

deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-backports main restricted universe multiverse

#deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-backports main restricted universe multiverse

deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-security main restricted universe multiverse

#deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-security main restricted universe multiverse source /etc/network/interfaces.d/*

sudo apt-get update

（4）sudo apt-get install openssh-server

（5）sudo vi /etc/network/interfaces

#The loopback network interface

auto lo
iface lo inet loopback

#The primary network interface

auto enp0s3
iface enp0s3 inet dhcp

auto enp0s8
iface enp0s8 inet static
address 192.168.0.101
netmask 255.255.255.0

sudo /etc/init.d/networking restart

重启，ifconfig查看，网卡启动成功，主机可以ping通

# 3 SSH 和用户添加

- 启动ssh服务

```
sudo service ssh start
```

- 添加用户（密码matrix)

```
sudo adduser neo
```

- 使neo成为sudo用户

```
sudo visudo
neo ALL=(ALL:ALL) ALL
```

- 换用cmd登录(方便复制粘贴)

```
ssh neo@192.168.0.101
```

# 4 Bind9

- 安装

```
sudo apt-get install bind9
```

- 配置1

```
sudo vi /etc/bind/named.conf.local
zone "oasis.com"{

    type master;

    file "/etc/bind/db.oasis.com";

//    allow-query-cache { any; };

    allow-query { any; };

    allow-update { any; };

//    recursion yes;

};


zone "0.168.192.in-addr.arpa"{

    type master;

    file "/etc/bind/db.192.oasis.com";

};
```

- 配置2

```
sudo vi /etc/bind/db.oasis.com
$TTL 604800

$ORIGIN oasis.com.

@ IN SOA oasis.com. root.oasis.com. (

2006080401 ; Serial

604800 ; Refresh

86400 ; Retry

2419200 ; Expire

604800 ) ; Negative Cache TTL

@ IN NS ns

@ IN A 192.168.0.101

mail IN A 192.168.0.101
web IN A 192.168.0.101
ns IN A 192.168.0.101
race IN A 192.168.0.101
shining IN A 192.168.0.101
game IN A 192.168.0.101
```

- 配置3

```
sudo vi /etc/bind/db.192.oasis.com 
$TTL 604800

@ IN SOA oasis.com. root.oasis.com. (

2006080401 ; Serial

604800 ; Refresh

86400 ; Retry

2419200 ; Expire

604800 ) ; Negative Cache TTL

;

@ IN NS oasis.com.

101 IN PTR mail.oasis.com.
101 IN PTR web.oasis.com.
101 IN PTR ns.oasis.com.
101 IN PTR race.oasis.com.
101 IN PTR shining.oasis.com.
101 IN PTR game.oasis.com.
```

- 重启bind9

  ```
  sudo vi /etc/resolv.conf
  ```

  ```
  # 附加在最前面
  nameserver 127.0.0.1
  ```

```
sudo /etc/init.d/bind9 restart
```

- 测试

```
# 正向
ping web.oasis.com
# 反向
host 192.168.0.101
```

配置网卡的dns：

配置dns为192.168.0.101

# 5 Nginx

- 安装

```
sudo apt install nginx
```

- 创建网站目录

```
sudo vi /etc/nginx/sites-available/default
sudo mkdir /var/www/race
sudo mkdir /var/www/shining
sudo mkdir /var/www/game
sudo chown -R www-data:www-data /var/www/race/
sudo chown -R www-data:www-data /var/www/shining/
sudo chown -R www-data:www-data /var/www/game/
```

- race网站主页

```
sudo vi /var/www/race/index.html
<h1> race.oasis.com </h1>
```

- 配置nginx

```
sudo cp /etc/nginx/sites-available/default /etc/nginx/sites-available/race
sudo cp /etc/nginx/sites-available/default /etc/nginx/sites-available/shining
sudo cp /etc/nginx/sites-available/default /etc/nginx/sites-available/game
```

**race、shining、game分别进行以下操作**

```
sudo vi /etc/nginx/sites-available/race
# 删掉原本的default_server
listen 80;
listen [::]:80;

root /var/www/race;
server_name race.oasis.com;


sudo vi /etc/nginx/sites-available/shining
# 删掉原本的default_server
listen 80;
listen [::]:80;

root /var/www/shining;
server_name shining.oasis.com;


udo vi /etc/nginx/sites-available/game
# 删掉原本的default_server
listen 80;
listen [::]:80;

root /var/www/game;
server_name game.oasis.com;
```

- 建立软连接

```
sudo ln -s /etc/nginx/sites-available/race /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/shining /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/game /etc/nginx/sites-enabled/
```

- 重启nginx

```
sudo systemctl restart nginx
```

- 测试

```
curl race.oasis.com
```

得到

```
<h1> race.oasis.com </h1>
```

说明配置成功

# 6 https加密

- 生成证书和密钥

```
sudo mkdir /etc/nginx/ssl
cd /etc/nginx/ssl
# 创建服务器私钥，命令会让你输入一个口令：
sudo openssl genrsa -des3 -out server.key 1024

# 创建签名请求的证书（CSR）：
sudo openssl req -new -key server.key -out server.csr

# 使用上面两个命令后，会要求输入，长度要大于4，记住输入的东西
# 不要切目录

sudo cp server.key server.key.org
sudo openssl rsa -in server.key.org -out server.key
# 输入刚才的东西

sudo openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt
```

- 配置nginx

```
sudo vi /etc/nginx/sites-available/race
```

添加（原来的保留，只是附加）

放在server外面，重写的一个server

```
server {

   listen 443;

   server_name race.oasis.com;

   ssl on;

   ssl_certificate /etc/nginx/ssl/server.crt;

   ssl_certificate_key /etc/nginx/ssl/server.key;

   ssl_session_timeout 5m;

   ssl_protocols TLSv1 TLSv1.1 TLSv1.2;

   ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;

   ssl_prefer_server_ciphers on;

   location / {

     root  /var/www/race;

     index index.php index.html index.htm;

   }

}
```

- 重启nginx

```
sudo systemctl restart nginx
```

参考

https://blog.csdn.net/weixin_35884835/article/details/52588157

https://www.jianshu.com/p/3811dc7f598b

# 7 PHP

- 安装

```
sudo apt-get install php7.0 php7.0-cli php7.0-common php7.0-fpm php7.0-gd php7.0-json php7.0-mbstring php7.0-mcrypt php7.0-mcrypt php7.0-mysql php7.0-opcache php7.0-readline php7.0-sqlite3
```

- 修改php.ini

```
sudo vi /etc/php/7.0/fpm/php.ini
# 直接加
cgi.fix_pathinfo = 0
```

- 修改www.conf

```
sudo vi /etc/php/7.0/fpm/pool.d/www.conf
```

直接加

```
listen.mode = 0666
```

- **重启！！！**

```
reboot
```

- 修改nginx里shining的配置

```
sudo vi /etc/nginx/sites-available/shining
```

server { } 里面的index添加index.php

```
index index.php index.html index.htm index.nginx-debian.html;
```

server { } 里面添加

```
location ~ \.php$ {
                include snippets/fastcgi-php.conf;
                fastcgi_pass unix:/run/php/php7.0-fpm.sock;
}
```

- 编写PHP网页界面

```
sudo vi /var/www/shining/index.php
<?php
phpinfo();
?>
```

- 重启nginx

```
sudo systemctl restart nginx
```

# 8 Java 、Tomcat 及 nginx代理

- 安装jdk

```
cd /tmp
wget http://172.18.5.74/download/jdk-8u112-linux-x64.tar.gz
sudo mkdir /usr/lib/java
sudo tar -C /usr/lib/java -xzf jdk-8u112-linux-x64.tar.gz
sudo vi  ~/.bashrc
```

添加

```
export JAVA_HOME=/usr/lib/java/jdk1.8.0_112
export JRE_HOME=${JAVA_HOME}/jre
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib
export PATH=${JAVA_HOME}/bin:$PATH
```

**重启**

```
reboot
```

测试

```
java -version
```

- 安装tomcat

```
cd /tmp
wget https://mirror.bit.edu.cn/apache/tomcat/tomcat-8/v8.5.61/bin/apache-tomcat-8.5.61.tar.gz
mkdir ~/tomcat
cp apache-tomcat-8.5.61.tar.gz ~/tomcat/
cd ~/tomcat/
tar -zxvf apache-tomcat-8.5.61.tar.gz
```

- 改环境变量

```
sudo vi /etc/profile
export TOMCAT_HOME=/home/neo/tomcat/apache-tomcat-8.5.61
export CATALINA_HOME=/home/neo/tomcat/apache-tomcat-8.5.61
```

- 启动Tomcat

```
cd ~/tomcat/apache-tomcat-8.5.61/bin
 ./startup.sh
```

- tomcat自启 （用户级别）

```
sudo vi ~/.bashrc
```

add

```
/home/neo/tomcat/apache-tomcat-8.5.61/bin/startup.sh
```

- 代理 


```
sudo vi /etc/nginx/sites-available/game

把原来的location /的要删除  
添加下面的在server里面
location / {
                # First attempt to serve request as file, then
                # as directory, then fall back to displaying a 404.
                proxy_pass http://192.168.0.101:8080;
                try_files $uri $uri/ =404;
        }
       
  location ~* \.(css|js|txt)$ {
        proxy_pass http://192.168.0.101:8080;
        expires 10d;
        }

location ~* \.(html|gif|jpg|jpeg|png|bmp|swf|ico|flv|mp3|wav|wmv|svg)$ {
        proxy_pass http://192.168.0.101:8080;
        expires 15d;
        access_log      off;
}

location ~* \.(htacess|tar.gz|tar|zip|sql)$ {
        return http://192.168.0.101:8080;
}

```

sudo service nginx restart

# 9 邮件

- 安装

```
sudo apt update
sudo DEBIAN_PRIORITY=low apt install postfix
```

- **General type of mail configuration?（一般邮件配置类型？）**：这个我们选择**Internet Site，**因为这符合我们的基础架构需求。
- **System mail name（系统邮件名称）**：这是用于在仅给出地址的帐户部分时构造有效电子邮件地址的基本域。 例如，我们服务器的主机名是`mail.oasis.com`，但我们可能希望将系统邮件名称设置为oasis.com`，以便给定用户名`user1`，Postfix将使用地址`user1@oasis.com。  oasis.com
- **（Root and postmaster mail recipient）root和邮件管理员**：这是Linux的帐户将被转发邮件的收件人是`root@`和`postmaster@`。使用您的主帐户。在我们的例子中，**neo**。
- **（Other destinations to accept mail for）接受邮件的其他目的地**：这定义了此Postfix实例将接受的邮件目的地。如果您需要添加此服务器负责接收来自其他域名的邮件，请在此处添加；默认情况下是可以正常工作的。
- **（Force synchronous updates on mail queue?）强制对邮件队列进行同步更新？**：由于您可能正在使用日志文件系统，因此请在此处选择**No**。
- **（Local networks）本地网络**：这是您的邮件服务器配置为中继邮件的网络列表。默认应适用于大多数方案。如果您选择修改它，请确保对网络范围有非常严格的限制
- **（Mailbox size limit）邮箱大小限制**：这可用于限制邮件的大小。设置成`0`后，就不会限制邮件大小了。
- **（Local address extension character）本地地址扩展字符**：这是可用于将地址的常规部分与扩展名（用于创建动态别名）分开的字符。
- **（Internet protocols to use）要使用的Internet协议**：我们会选择`all`。

```
sudo apt install mailutils
```

- 发邮件

```
echo 'it is only a test' | mail -s "test eamil" neo@oasis.com 
```

# 10 MySQL

- 安装 密码设为 oracle

```
sudo apt install mysql-server
```

- 改端口

```
sudo vi /etc/mysql/mysql.conf.d/mysqld.cnf
port = 3366
sudo systemctl restart mysql
```

- 创建数据库

```
mysql -u root -p
```

输入密码 oracle , 进入mysql 命令行界面

```
mysql> CREATE DATABASE zion;
mysql> quit
```

- 测试

```
mysql -u root -P 3366 -h localhost -D zion -poracle
```

看到如下界面说明成功了

```
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 4
Server version: 5.7.32-0ubuntu0.16.04.1 (Ubuntu)

Copyright (c) 2000, 2020, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> 
mysql> quit
```

# 11 防火墙

- 启动防火墙

```
sudo ufw enable
```

- 添加规则

```
sudo ufw allow 22
sudo ufw allow 25
sudo ufw allow 80
sudo ufw allow 443
sudo ufw allow 110
sudo ufw allow 143
```

- 查看规则

```
sudo ufw status numbered
```

# 11 RAID 1

- 添加2块1GB的虚拟硬盘
- 查看磁盘

```
lsblk -o NAME,SIZE,FSTYPE,TYPE,MOUNTPOINT
```

- 创建md0 sdb sdc根据实际情况换

```
sudo mdadm --create --verbose /dev/md0 --level=1 --raid-devices=2 /dev/sdb /dev/sdc
```

- 挂载（若挂载失败报错，可能是没有格式化，可以先格式化mkfs.ext4 /dev/md0在mount上去）

```
sudo mkdir -p /mnt/md1
sudo mount /dev/md0 /mnt/md1
```

查看

```
df -h -x devtmpfs -x tmpfs
```

- 开机自动挂载

```
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
sudo update-initramfs -u
echo '/dev/md0 /mnt/md1 ext4 defaults,nofail,discard 0 0' | sudo tee -a /etc/fstab
```

# 12 交卷准备

- 重启

```
reboot
```

- 修改本地DNS

```
sudo vi /etc/resolv.conf
```

加在最前面

```
nameserver 127.0.0.1
```

