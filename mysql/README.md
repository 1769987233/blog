## 安装mysql5.7

```yaml
#====================下载镜像================================
docker pull mysql:5.7.34

#================复制配置文件到宿主机，主从都适用=================
docker rm -f mysql5.7_master
docker rm -f mysql5.7_slave
docker run -d --name mysql5.7_master \
-p 3306:3306 \
mysql:5.7.34

#docker exec -it mysql5.7_master /bin/bash

mkdir -p /home/mysql/conf
mkdir -p /home/mysql/log
mkdir -p /home/mysql/data
rm -rf /home/mysql/conf/*
rm -rf /home/mysql/log/* 
rm -rf /home/mysql/data/*

# 复制配置文件到宿主机
docker cp mysql5.7_master:/etc/mysql /home/mysql/conf 
docker cp mysql5.7_master:/var/log /home/mysql/log 
docker cp mysql5.7_master:/var/lib /home/mysql/data

#======================宿主机=======================
# 查看挂载目录
[root@yangzaigang mysql]# ll
total 16
drwxr-xr-x 2 root root 4096 Apr 20 02:57 conf.d
lrwxrwxrwx 1 root root   24 Apr 20 02:57 my.cnf -> /etc/alternatives/my.cnf
-rw-r--r-- 1 root root  839 Aug  3  2016 my.cnf.fallback
-rw-r--r-- 1 root root 1200 Mar 26 15:10 mysql.cnf
drwxr-xr-x 2 root root 4096 Apr 20 02:57 mysql.conf.d
# 宿主机
[root@yangzaigang mysql]# ll /etc/alternatives/my.cnf
-rw-r--r-- 1 root root 0 May 11 16:22 /etc/alternatives/my.cnf

#====================容器内==============================
root@b28cf0d21007:/etc/mysql# ls -l
total 16
drwxr-xr-x 2 root root 4096 Apr 19 18:57 conf.d
lrwxrwxrwx 1 root root   24 Apr 19 18:57 my.cnf -> /etc/alternatives/my.cnf
-rw-r--r-- 1 root root  839 Aug  3  2016 my.cnf.fallback
-rw-r--r-- 1 root root 1200 Mar 26 07:10 mysql.cnf
drwxr-xr-x 2 root root 4096 Apr 19 18:57 mysql.conf.d

root@b28cf0d21007:/etc/mysql# ls -l /etc/alternatives/my.cnf
lrwxrwxrwx 1 root root 20 Apr 19 18:57 /etc/alternatives/my.cnf -> /etc/mysql/mysql.cnf

#======================宿主机=======================
rm -rf /etc/alternatives/my.cnf
#vi cat tee 登向my.cnf写入内容，就会产生/etc/alternatives/my.cnf 这个文件

# 删除容器
docker rm -f mysql5.7_master  

#====================主=============================
# 清空配置文件，别删除这个配置文件
> /home/mysql/conf/mysql/my.cnf 
#============
tee /home/mysql/conf/mysql/mysql.cnf <<-'EOF'
[client]
default-character-set=utf8mb4
[mysql]
default-character-set=utf8mb4
[mysqld]
character-set-server=utf8mb4 
collation-server=utf8mb4_general_ci
sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION
!includedir /etc/mysql/conf.d/
!includedir /etc/mysql/mysql.conf.d/
EOF
#==============
tee /home/mysql/conf/mysql/mysql.conf.d/mysqld.cnf <<-'EOF'
[mysqld]
pid-file	= /var/run/mysqld/mysqld.pid
socket		= /var/run/mysqld/mysqld.sock
datadir		= /var/lib/mysql
#log-error	= /var/log/mysql/error.log
symbolic-links=0
bind-address = 0.0.0.0
server-id=5
log-bin=mysql-bin
binlog-ignore-db = mysql
binlog_ignore_db = information_schema
binlog_ignore_db = performation_schema
binlog_ignore_db = sys
EOF

cat /home/mysql/conf/mysql/mysql.conf.d/mysqld.cnf

#===============从，使用relay-log=mysql-relay =========================
#==============多从时，server-id不一样 ========================
# 清空配置文件，别删除这个配置文件
> /home/mysql/conf/mysql/my.cnf 
#============
tee /home/mysql/conf/mysql/mysql.cnf <<-'EOF'
[client]
default-character-set=utf8mb4
[mysql]
default-character-set=utf8mb4
[mysqld]
character-set-server=utf8mb4 
collation-server=utf8mb4_general_ci
sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION
!includedir /etc/mysql/conf.d/
!includedir /etc/mysql/mysql.conf.d/
EOF
#==============
tee /home/mysql/conf/mysql/mysql.conf.d/mysqld.cnf <<-'EOF'
[mysqld]
pid-file	= /var/run/mysqld/mysqld.pid
socket		= /var/run/mysqld/mysqld.sock
datadir		= /var/lib/mysql
#log-error	= /var/log/mysql/error.log
symbolic-links=0
bind-address = 0.0.0.0
server-id=7
relay-log=mysql-relay
binlog-ignore-db = mysql
binlog_ignore_db = information_schema
binlog_ignore_db = performation_schema
binlog_ignore_db = sys
EOF

#==================docker方式启动=======================
#==================主mysql============
docker rm -f mysql5.7_master
docker run -d \
--name mysql5.7_master \
--net=host \
-e MYSQL_ROOT_PASSWORD=Leyun@123456 \
-v /home/mysql/conf/mysql:/etc/mysql \
-v /home/mysql/log/mysql:/var/log/mysql \
-v /home/mysql/data/mysql:/var/lib/mysql \
mysql:5.7 \
--lower_case_table_names=1

#=================从mysql===============
docker run -d \
--name mysql5.7_slave \
--net=host \
--restart=always \
-e MYSQL_ROOT_PASSWORD=Leyun@123456 \
-v /home/mysql/conf/mysql:/etc/mysql \
-v /home/mysql/log/mysql:/var/log/mysql \
-v /home/mysql/datamysql:/var/lib/mysql \
mysql:5.7 \
--lower_case_table_names=1

#====================docker-compose方式启动===================
#============主==============
tee docker-compose.yml <<-'EOF'
version: "3.8"
services:
  mysql:
    image: mysql:5.7
    container_name: mysql5.7_master
    network_mode: "host"
    restart: always
    privileged: true
    environment:
      - MYSQL_ROOT_PASSWORD=Leyun@123456
      - TZ=Asia/shanghai
    volumes:
      - /home/mysql/conf/mysql:/etc/mysql
      - /home/mysql/log/mysql:/var/log/mysql
      - /home/mysql/data/mysql:/var/lib/mysql
    command: 
      --default-authentication-plugin=mysql_native_password
      --lower_case_table_names=1    
EOF
echo "================"
cat docker-compose.yml

#=============从===========
tee docker-compose.yml <<-'EOF'
version: "3.8"
services:
  mysql:
    image: mysql:5.7
    container_name: mysql5.7_slave
    network_mode: "host"
    restart: always
    privileged: true
    environment:
      - MYSQL_ROOT_PASSWORD=Leyun@123456
      - TZ=Asia/shanghai
    volumes:
      - /home/mysql/conf/mysql:/etc/mysql
      - /home/mysql/log/mysql:/var/log/mysql
      - /home/mysql/data/mysql:/var/lib/mysql
    command: 
      --default-authentication-plugin=mysql_native_password
      --lower_case_table_names=1    
EOF
echo "==============="
cat docker-compose.yml
```

## 配置主从

```yaml
#==========================配置主从============================
# 进入容器
docker exec -it mysql5.7_master /bin/bash
docker exec -it mysql5.7_slave /bin/bash

# 进入数据库
mysql -uroot -p
Leyun@123456

#查看权限信息
use mysql;
select host, user, authentication_string, plugin from user;

#查看user表的root用户Host字段是localhost，说明root用户只能本地登录，现在把他改成远程登录
#把root的权限更新
#update user set host='%' where user='root';

# 创建用户
CREATE USER 'slave'@'%' IDENTIFIED WITH mysql_native_password BY 'Leyun@123456';
CREATE USER 'slave'@'*' IDENTIFIED WITH mysql_native_password BY 'Leyun@123456';
#分配权限
#GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' WITH GRANT OPTION;
GRANT REPLICATION SLAVE ON *.* TO 'root'@'%';
GRANT REPLICATION SLAVE ON *.* TO 'slave'@'%';

#刷新权限
flush privileges;  

# 获取主（从）节点当前binary log文件名和位置（position）
show master status;

# 主节点配置
stop slave;
reset slave; 
CHANGE MASTER TO
MASTER_HOST='10.10.1.16',
MASTER_USER='slave',
MASTER_PASSWORD='Leyun@123456',
MASTER_PORT=3306,
MASTER_LOG_FILE='mysql-bin.000010',
MASTER_LOG_POS=419011;
start slave;
show slave status \G

```



## keepalived+双主热备

```yaml
#==========双主热备，只有server-id不一样================
cat > /home/mysql/conf/mysql/my.cnf << EOF
[client]
default-character-set=utf8mb4
[mysql]
default-character-set=utf8mb4
[mysqld]
character-set-server=utf8mb4 
collation-server=utf8mb4_general_ci
EOF
cat > /home/mysql/conf/mysql/mysql.conf.d/mysqld.cnf << EOF
[mysqld]
pid-file	= /var/run/mysqld/mysqld.pid
socket		= /var/run/mysqld/mysqld.sock
datadir		= /var/lib/mysql
#log-error	= /var/log/mysql/error.log
symbolic-links=0
bind-address = 0.0.0.0
server-id=11
log-bin=mysql-bin
relay-log=mysql-relay
binlog-ignore-db = mysql
binlog_ignore_db = information_schema
binlog_ignore_db = performation_schema
binlog_ignore_db = sys
EOF

cat /home/mysql/conf/mysql/mysql.conf.d/mysqld.cnf
```

----

```yaml
#==========================安装keepalived=========================
# 安装keepalived
yum install keepalived 1.3.5
# 自启动
systemctl enable keepalived

#===========================编写脚本==================================
# 编写脚本监控mysql是否正常运行
# 会自动生成文件
vi /etc/keepalived/check_mysql.sh 
#!/bin/bash
MYSQL_PING=$(docker exec mysql5.7_master mysqladmin -h127.0.0.1 -uroot -pLeyun@123456 ping 2>/dev/null)
MYSQL_OK="mysqld is alive"
if [[ "$MYSQL_PING" != "$MYSQL_OK" ]];then
   echo "mysql is not running."
   killall keepalived
else
   echo "mysql is running"
fi

# 添加可自行权限
chmod +x /etc/keepalived/check_mysql.sh
cat /etc/keepalived/check_mysql.sh

#=========================备份配置文件===============================
cp /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.bak

#============================主=====================================
# 编写keepalived配置文件
cat > /etc/keepalived/keepalived.conf <<EOF
! Configuration File for keepalived
global_defs {
}
vrrp_script check_mysql {
  script "/etc/keepalived/check_mysql.sh"
  interval 2
}
vrrp_instance VI_1 {
  state MASTER
  interface enp0s3
  virtual_router_id 51
  priority 100
  advert_int 1
  nopreempt
  authentication {
    auth_type PASS
    auth_pass 1111
  }
  virtual_ipaddress {
    10.10.5.254
  }
  track_script {
    check_mysql
  }
}
EOF

#=========================从=======================================
cat > /etc/keepalived/keepalived.conf <<EOF
! Configuration File for keepalived
global_defs {
}
vrrp_script check_mysql {
  script "/etc/keepalived/check_mysql.sh"
  interval 2
}
vrrp_instance VI_1 {
  state BACKUP
  interface enp0s3
  virtual_router_id 51
  priority 90
  advert_int 1
  authentication {
    auth_type PASS
    auth_pass 1111
  }
  virtual_ipaddress {
    10.10.5.254
  }
  track_script {
    check_mysql
  }
}
EOF

#===================启动keepalived====================
systemctl start keepalived
systemctl restart keepalived

#===================查看IP地址============================
[root@localhost keepalived]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:ec:7c:d2 brd ff:ff:ff:ff:ff:ff
    inet 10.10.5.128/20 brd 10.10.15.255 scope global noprefixroute dynamic enp0s3
       valid_lft 2891sec preferred_lft 2891sec
    inet 10.10.5.254/32 scope global enp0s3
       valid_lft forever preferred_lft forever
    inet6 fe80::ceb1:fa58:e3d6:8fe7/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever

```

----

```yaml
#======================两台主机ping VIP=================================
# 主
[root@localhost ~]# ip a | grep "inet 10"
    inet 10.10.5.128/20 brd 10.10.15.255 scope global noprefixroute dynamic enp0s3
    inet 10.10.5.254/32 scope global enp0s3
[root@localhost ~]# ping 10.10.5.254
PING 10.10.5.254 (10.10.5.254) 56(84) bytes of data.
64 bytes from 10.10.5.254: icmp_seq=1 ttl=64 time=0.055 ms
64 bytes from 10.10.5.254: icmp_seq=2 ttl=64 time=0.050 ms
# 从
[root@localhost ~]# ip a | grep "inet 10"
    inet 10.10.5.130/20 brd 10.10.15.255 scope global noprefixroute dynamic enp0s3
[root@localhost ~]# ping 10.10.5.254
PING 10.10.5.254 (10.10.5.254) 56(84) bytes of data.
64 bytes from 10.10.5.254: icmp_seq=1 ttl=64 time=0.507 ms
64 bytes from 10.10.5.254: icmp_seq=2 ttl=64 time=0.397 ms

#=====================停掉主mysql=========================================
# 停掉主mysql
docker stop mysql5.7_master

# 在主上查看IP
[root@localhost ~]# ip a | grep "inet 10"
    inet 10.10.5.128/20 brd 10.10.15.255 scope global noprefixroute dynamic enp0s3

# 在从查看IP，IP地址已经漂移过来了
[root@localhost ~]# ip a | grep "inet 10"
    inet 10.10.5.130/20 brd 10.10.15.255 scope global noprefixroute dynamic enp0s3
    inet 10.10.5.254/32 scope global enp0s3
    
# 启动主mysql，启动keepalived，虚拟IP会漂移过来（配置为抢占模式）
docker start mysql5.7_master
systemctl start keepalived
[root@localhost ~]# ip a | grep "inet 10"
    inet 10.10.5.128/20 brd 10.10.15.255 scope global noprefixroute dynamic enp0s3
    inet 10.10.5.254/32 scope global enp0s3
[root@localhost ~]# 
```



```
> /home/mysql2/conf/mysql/my.cnf 
#============
tee /home/mysql2/conf/mysql/mysql.cnf <<-'EOF'
[client]
default-character-set=utf8mb4
[mysql]
default-character-set=utf8mb4
[mysqld]
character-set-server=utf8mb4 
collation-server=utf8mb4_general_ci
sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION
!includedir /etc/mysql/conf.d/
!includedir /etc/mysql/mysql.conf.d/
EOF

#docker restart mysql5.7_master
docker restart mysql5.7_slave


cat /home/mysql2/conf/mysql/mysql.conf.d/mysqld.cnf
```

