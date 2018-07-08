## Win环境部署和测试前后端程序

1、[下载maven](http://maven.apache.org/download.cgi)并解压

2、设置环境变量PATH

3、修改conf/settings.xml文件，设置jar包存放目录（可选）

`<localRepository>D:\maven-repo</localRepository>`

4、安装Java，[设置JAVA_HOME](https://www.cnblogs.com/boringwind/p/8001300.html)，这里必须要设置，不然mvn构建的时候会找不到JDK

5、下载安装eclipse，设置maven的配置文件路径

`Window -> Preferences -> Maven -> User Settings`

6、[下载renren-fast](http://www.renren.io/)项目，将项目导入到eclipse
```
# git clone https://gitee.com/renrenio/renren-fast.git
```

7、下载安装xampp，启动mysql

8、启动apache，通过phpmyadmin修改数据库root密码

9、下载安装datagrip

10、启动datagrip，连接数据库，复制eclipse上的mysql.sql里的全部内容

11、在datagrip创建一个schema命名为renren_fast，然后在renren_fast上创建一个console，将刚才复制的sql内容粘贴到console上，全选并执行

12、给eclipse安装sts插件

`Help -> Eclipse Marketplace`

13、在eclipse上修改数据库连接信息，用springboot app运行工程，这样后端工程就启动了

14、查看swagger，并测试接口

http://localhost:8080/renren-fast/swagger/index.html

15、下载前端代码
```
# git clone https://github.com/daxiongYang/renren-fast-vue.git
```

16、下载并安装nodejs

17、进入前端项目目录，安装依赖包
```
# cd renren-fast-vue/
# npm install
```

18、运行前端项目
```
# npm run dev
```

19、查看前端页面，并尝试登录，默认账户admin/admin

http://localhost:8001

---

## Docker虚拟机

- 隔离性
- 轻量级虚拟机
- 共用一个Linux内核
- 硬件资源占用较小

1、安装docker
```
# yum install -y docker-ce
```

2、配置加速器

https://www.daocloud.io/mirror#accelerator-doc

## 使用docker部署pxc集群

- PXC Percona XtraDB Cluster
- 集群中的每个实例都是可读可写的
- 数据强一致性，同步复制，事务在所有节点要么同时提交，要么不提交

1、[下载PXC镜像](https://hub.docker.com/r/percona/percona-xtradb-cluster/)
```
# docker pull percona/percona-xtradb-cluster
```

2、重命名镜像
```
# docker tag percona/percona-xtradb-cluster pxc
```

3、创建net1网络
```
# docker network create --subnet=172.18.0.0/16 net1
```

4、创建5个数据卷。由于通过映射宿主机目录到容器中时，PXC无法方式正常启动。这里使用docker volume来创建数据卷
```
# docker volume create v1
# docker volume create v2
# docker volume create v3
# docker volume create v4
# docker volume create v5
```

5、创建备份数据卷（用于热备份数据）
```
# docker volume create backup
```

6、创建5节点的PXC集群。注意，每个pxc容器创建之后，因为要执行PXC的初始化和加入集群等工作，耐心等待1分钟左右，再用客户端连接MySQL。另外，必须等第一个MySQL节点启动成功，用MySQL客户端能连接上之后，再去创建其他MySQL节点。

```
# docker run -d -p 3306:3306 -e MYSQL_ROOT_PASSWORD=abc123456 -e CLUSTER_NAME=PXC -e XTRABACKUP_PASSWORD=abc123456 \
    -v v1:/var/lib/mysql -v backup:/data --privileged --name=node1 --net=net1 --ip 172.18.0.2 pxc
# docker run -d -p 3307:3306 -e MYSQL_ROOT_PASSWORD=abc123456 -e CLUSTER_NAME=PXC -e XTRABACKUP_PASSWORD=abc123456 \
    -e CLUSTER_JOIN=node1 -v v2:/var/lib/mysql -v backup:/data --privileged --name=node2 \
    --net=net1 --ip 172.18.0.3 pxc
# docker run -d -p 3308:3306 -e MYSQL_ROOT_PASSWORD=abc123456 -e CLUSTER_NAME=PXC -e XTRABACKUP_PASSWORD=abc123456 \
    -e CLUSTER_JOIN=node1 -v v3:/var/lib/mysql --privileged --name=node3 --net=net1 --ip 172.18.0.4 pxc
# docker run -d -p 3309:3306 -e MYSQL_ROOT_PASSWORD=abc123456 -e CLUSTER_NAME=PXC -e XTRABACKUP_PASSWORD=abc123456 \
    -e CLUSTER_JOIN=node1 -v v4:/var/lib/mysql --privileged --name=node4 --net=net1 --ip 172.18.0.5 pxc
# docker run -d -p 3310:3306 -e MYSQL_ROOT_PASSWORD=abc123456 -e CLUSTER_NAME=PXC -e XTRABACKUP_PASSWORD=abc123456 \
    -e CLUSTER_JOIN=node1 -v v5:/var/lib/mysql -v backup:/data --privileged --name=node5 --net=net1 \
    --ip 172.18.0.6 pxc
```

7、下载haproxy镜像
```
# docker pull haproxy
```

8、宿主机上编写Haproxy配置文件
```
# vim /home/soft/haproxy/haproxy.cfg

  global
    #工作目录
    chroot /usr/local/etc/haproxy
    #日志文件，使用rsyslog服务中local5日志设备（/var/log/local5），等级info
    log 127.0.0.1 local5 info
    #守护进程运行
    daemon

  defaults
    log  global
    mode  http
    #日志格式
    option  httplog
    #日志中不记录负载均衡的心跳检测记录
    option  dontlognull
    #连接超时（毫秒）
    timeout connect 5000
    #客户端超时（毫秒）
    timeout client  50000
    #服务器超时（毫秒）
    timeout server  50000

  #监控界面
  listen  admin_stats
  #监控界面的访问的IP和端口
  bind  0.0.0.0:8888
  #访问协议
  mode http
  #URI相对地址
  stats uri   /dbs
  #统计报告格式
  stats realm     Global\ statistics
  #登陆帐户信息
  stats auth  admin:abc123456

  #数据库负载均衡
  listen  proxy-mysql
  #访问的IP和端口
  bind  0.0.0.0:3306
  #网络协议
  mode tcp
  #负载均衡算法（轮询算法）
  #轮询算法：roundrobin
  #权重算法：static-rr
  #最少连接算法：leastconn
  #请求源IP算法：source
  balance  roundrobin

  #日志格式
  option  tcplog

  #在MySQL中创建一个没有权限的haproxy用户，密码为空。Haproxy使用这个账户对MySQL数据库心跳检测
  option  mysql-check user haproxy
  server  MySQL_1 172.18.0.2:3306 check weight 1 maxconn 2000
  server  MySQL_2 172.18.0.3:3306 check weight 1 maxconn 2000
  server  MySQL_3 172.18.0.4:3306 check weight 1 maxconn 2000
  server  MySQL_4 172.18.0.5:3306 check weight 1 maxconn 2000
  server  MySQL_5 172.18.0.6:3306 check weight 1 maxconn 2000

  #使用keepalive检测死链
  option  tcpka
```

为Haproxy创建健康检测用户
```
create user 'haproxy'@'%' identified by '';
```

9、创建两个haproxy容器，并启动服务
```
# docker run -it -d -p 4001:8888 -p 4002:3306 -v /home/soft/haproxy:/usr/local/etc/haproxy \
    --name h1 --privileged --net=net1 --ip 172.18.0.7 haproxy
# docker exec -it h1 bash
# haproxy -f /usr/local/etc/haproxy/haproxy.cfg
```
```
# docker run -it -d -p 4003:8888 -p 4004:3306 -v /home/soft/haproxy:/usr/local/etc/haproxy \
    --name h2 --privileged --net=net1 --ip 172.18.0.8 haproxy
# docker exec -it h2 bash
# haproxy -f /usr/local/etc/haproxy/haproxy.cfg
```

10、Haproxy容器内安装Keepalived，设置虚拟IP
```
进入h1容器
# docker exec -it h1 bash
# apt-get update
# apt-get install vim keepalived
# vim /etc/keepalived/keepalived.conf
# service keepalived start

宿主机执行ping命令
# ping 172.18.0.201
```
```
进入h2容器
# docker exec -it h2 bash
# apt-get install vim keepalived
# vim /etc/keepalived/keepalived.conf
# service keepalived start

宿主机执行ping命令
# ping 172.18.0.201
```

11、宿主机安装Keepalived
```
# yum -y install keepalived
# vim /etc/keepalived/keepalived.conf
# service keepalived start
```

http://192.168.135.201:8888/dbs

Xtra-Backup的优势

- 备份过程不锁表
- 备份过程不会打断正在执行的事务
- 支持压缩节省磁盘空间和流量

一周一次全量备份，一天一次增量备份

12、热备份数据
```
# docker exec -it node1 bash
# apt-get update
a# pt-get install percona-xtrabackup-24

全量热备
# innobackupex --user=root --password=abc123456 /data/backup/full
```

13、冷还原数据，数据库只能冷还原。停止其余4个节点，并删除节点
```
# docker stop node2
# docker stop node3
# docker stop node4
# docker stop node5
# docker rm node2
# docker rm node3
# docker rm node4
# docker rm node5
```

node1容器中删除MySQL的数据
```
rm -rf /var/lib/mysql/*
```

清空事务
```
innobackupex --user=root --password=abc123456 --apply-back /data/backup/full/2018-04-15_05-09-07/
```

还原数据
```
innobackupex --user=root --password=abc123456 --copy-back  /data/backup/full/2018-04-15_05-09-07/
```

重新创建其余4个节点，组建PXC集群

## 部署RedisCluster集群

高速缓存
- 使用内存保存数据，读写速度远超硬盘
- 高速缓存可以减少IO操作，降低IO压力
- 可以提供10W次每秒的读写
- 新浪微博搭建了世界上最大的Redis集群

常见产品
- RedisCluster：官方推荐，没有中心节点
- Codis：中间件产品，有中心节点
- Twemproxy：中间件产品，有中心节点

1、下载Redis镜像
```
# docker pull yyyyttttwwww/redis
```

2、创建net2网段
```
# docker network create --subnet=172.19.0.0/16 net2
```

3、创建6节点Redis容器
```
# docker run -it -d --name r1 -p 5001:6379 --net=net2 --ip 172.19.0.2 redis bash
# docker run -it -d --name r2 -p 5002:6379 --net=net2 --ip 172.19.0.3 redis bash
# docker run -it -d --name r3 -p 5003:6379 --net=net2 --ip 172.19.0.4 redis bash
# docker run -it -d --name r4 -p 5004:6379 --net=net2 --ip 172.19.0.5 redis bash
# docker run -it -d --name r5 -p 5005:6379 --net=net2 --ip 172.19.0.6 redis bash
# docker run -it -d --name r6 -p 5006:6379 --net=net2 --ip 172.19.0.7 redis bash
```

4、启动6个实例的redis服务
```
docker exec -it r1 bash
cp /home/redis/redis.conf /usr/redis/redis.conf
cd /usr/redis/src
./redis-server ../redis.conf

docker exec -it r2 bash
cp /home/redis/redis.conf /usr/redis/redis.conf
cd /usr/redis/src
./redis-server ../redis.conf

docker exec -it r3 bash
cp /home/redis/redis.conf /usr/redis/redis.conf
cd /usr/redis/src
./redis-server ../redis.conf

docker exec -it r4 bash
cp /home/redis/redis.conf /usr/redis/redis.conf
cd /usr/redis/src
./redis-server ../redis.conf

docker exec -it r5 bash
cp /home/redis/redis.conf /usr/redis/redis.conf
cd /usr/redis/src
./redis-server ../redis.conf

docker exec -it r6 bash
cp /home/redis/redis.conf /usr/redis/redis.conf
cd /usr/redis/src
./redis-server ../redis.conf
```

5、创建Cluster集群

在r1节点上
```
cd /usr/redis/src
mkdir -p ../cluster
cp redis-trib.rb ../cluster/
cd ../cluster

创建Cluster集群
./redis-trib.rb create --replicas 1 172.19.0.2:6379 172.19.0.3:6379 172.19.0.4:6379 172.19.0.5:6379 172.19.0.6:6379 172.19.0.7:6379

# redis-cli -c
> cluster nodes
```

---

## 打包部署后端项目

1、进入人人开源后端项目，执行打包（分别修改配置文件，更改端口，打包三次生成三个JAR文件）
```
# mvn clean install -Dmaven.test.skip=true

clean：清除之前的JAR文件
install：打包到本地
-Dmaven.test.skip=true：跳过测试代码
```
```
redis配置

cluster:
  nodes:
  - 172.19.0.2:6379
  - 172.19.0.3:6379
  - 172.19.0.4:6379
  - 172.19.0.5:6379
  - 172.19.0.6:6379
  - 172.19.0.7:6379
```

2、下载Java镜像
```
# docker pull java
```

3、创建3节点Java容器
```
docker volume create j1
docker run -it -d --name j1 -v j1:/home/soft --net=host java
docker exec -it j1 bash
nohup java -jar /home/soft/renren-fast.jar &
```
```
docker volume create j2
docker run -it -d --name j2 -v j2:/home/soft --net=host java
docker exec -it j2 bash
nohup java -jar /home/soft/renren-fast.jar &
```
```
docker volume create j3
docker run -it -d --name j3 -v j3:/home/soft --net=host java
docker exec -it j3 bash
nohup java -jar /home/soft/renren-fast.jar &
```

尝试访问后端API服务

http://192.168.135.170:6001/renren-fast/swagger/index.html

4、下载Nginx镜像。（nginx是性能非常出色的反代服务器，最大可支持8W/S的并发访问）
```
# docker pull nginx
```

5、创建两个nginx实例
```
# docker run -it -d --name n1 -v /home/n1/nginx.conf:/etc/nginx/nginx.conf --net=host --privileged nginx
# docker run -it -d --name n2 -v /home/n2/nginx.conf:/etc/nginx/nginx.conf --net=host --privileged nginx
```

6、在Nginx容器安装Keepalived
```
# docker exec -it n1 bash
# apt-get update
# apt-get install vim keepalived
# vim /etc/keepalived/keepalived.conf
# service keepalived start
```
```
# docker exec -it n2 bash
# apt-get update
# apt-get install vim keepalived
# vim /etc/keepalived/keepalived.conf
# service keepalived start
```
http://192.168.135.202:6201/renren-fast/swagger/index.html

---

## 打包部署前端项目

修改npm源

http://npm.taobao.org/

1、在前端项目路径下执行打包指令
```
# npm run build
```

2、将dist目录的文件拷贝到宿主机的`/home/fn1/renren-vue、/home/fn2/renren-vue、/home/fn3/renren-vue`目录下面

3、创建3节点的Nginx实例，部署前端项目
```
docker run -it -d --name fn1 -v /home/fn1/nginx.conf:/etc/nginx/nginx.conf -v /home/fn1/renren-vue:/home/fn1/renren-vue --privileged --net=host nginx
docker run -it -d --name fn2 -v /home/fn2/nginx.conf:/etc/nginx/nginx.conf -v /home/fn2/renren-vue:/home/fn2/renren-vue --privileged --net=host nginx
docker run -it -d --name fn3 -v /home/fn3/nginx.conf:/etc/nginx/nginx.conf -v /home/fn3/renren-vue:/home/fn3/renren-vue --privileged --net=host nginx
```

http://192.168.135.170:6501
http://192.168.135.170:6502
http://192.168.135.170:6503

4、配置负载均衡
```
docker run -it -d --name ff1 -v /home/ff1/nginx.conf:/etc/nginx/nginx.conf --net=host --privileged nginx
docker run -it -d --name ff2 -v /home/ff2/nginx.conf:/etc/nginx/nginx.conf --net=host --privileged nginx
```

http://192.168.135.170:6601
http://192.168.135.170:6602

5、配置高可用
```
docker exec -it ff1 bash
apt-get update
apt-get install vim keepalived
vim /etc/keepalived/keepalived.conf
service keepalived start
```
```
docker exec -it ff2 bash
apt-get update
apt-get install vim keepalived
vim /etc/keepalived/keepalived.conf
service keepalived start
```

http://192.168.135.203:6701


## 安装portainer

1、安装docker
```
# yum install -y docker
```

2、修改启动选项
```
# vim /etc/sysconfig/docker
OPTIONS='-H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock'
```

3、启动docker
```
# systemctl start docker
```

4、拉取portainer镜像
```
# docker pull portainer/portainer
```

5、启动portainer
```
# docker run -d -p 9000:9000 --name portainer portainer/portainer -H tcp://192.168.135.171:2375
```

## Docker Swarm

1、创建swarm集群
```
# docker swarm init
```

2、加入swarm集群
```
# docker swarm join-token manager
# docker swarm join-token worker
```

3、查看集群内的节点，只能在manager节点执行
```
# docker node ls
```

4、Swarm集群自带一个叫ingress的overlay共享网络
```
# docker network ls
```

5、创建共享网络
```
# docker network create -d overlay --attachable swarm_test
# docker network ls
```

6、退出集群
```
a、manager节点需要先降级为worker
# docker node demote

b、退出集群，worker节点上直接leave即可
# docker swarm leave

在manager节点上可以强行退出，不过仅限于集群将不再使用的情况
# docker swarm leave --force

c、将节点从集群中删除
# docker node rm NodeID
```