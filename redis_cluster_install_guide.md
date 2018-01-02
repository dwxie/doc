# Redis Cluster Install Guide

---

## 1. 环境

* redis 3.0.6
* Ubuntu 16.04.2 LTS


## 2. redis cluster原理

redis从3.0之后版本支持redis-cluster集群，Redis-Cluster采用无中心结构，每个节点保存数据和整个集群状态,每个节点都和其他所有节点连接。
其redis-cluster架构图如下：

![redis cluster架构](img/redis_cluster.jpg)

其结构特点：

  1. 所有的redis节点彼此互联(PING-PONG机制),内部使用二进制协议优化传输速度和带宽。
  2. 节点的fail是通过集群中超过半数的节点检测失效时才生效。
  3. 客户端与redis节点直连，不需要中间proxy层。客户端不需要连接集群所有节点，连接集群中任何一个可用节点即可。
  4. redis-cluster把所有的物理节点映射到[0-16383]slot上（不一定是平均分配），cluster负责维护node<->slot<->value。
  4. Redis集群预分好16384个桶，当需要在Redis集群中放置一个key-value时，根据 CRC16(key) mod 16384的值，决定将一个key放到哪个桶中。

### redis cluster节点分配

假设我们是三个主节点分别是：A，B，C三个节点，它们可以是一台机器上的三个端口，也可以是三台不同的服务器。那么，采用哈希槽（hash slot）的方式来分配16384个slot的话，三个节点分别承担的slot区间是：

  * 节点A覆盖0-5460
  * 节点B覆盖5461-10922
  * 节点C覆盖10923-16383

如果存入一个值，按照redis cluster哈希槽的算法：CRC16('key')%16384=6782。那么就会把这个key的存储分配到B上了。同样，当我连接(A,B,C)任何一个节点想获取'key'这个key时，也会按照这样的算法，然后内部跳转到B节点上获取数据。
	  
### redis cluster主从模式

redis cluster为了保证数据的高可用性，加入了主从模式，一个主节点对应一个或多个从节点，主节点提供数据存取，从节点则是从主节点拉取数据备份，当这个主节点挂掉后，就会有这个从节点选取一个来充当主节点，从而保证集群不会挂掉。

上面那个例子里, 集群有ABC三个主节点, 如果这3个节点都没有加入从节点，如果B挂掉了，我们就无法访问整个集群了。A和C的slot也无法访问。

所以我们在集群建立的时候，一定要为每个主节点都添加了从节点, 比如像这样, 集群包含主节点A、B、C, 以及从节点A1、B1、C1, 那么即使B挂掉系统也可以继续正确工作。

B1节点替代了B节点，所以Redis集群将会选择B1节点作为新的主节点，集群将会继续正确地提供服务。 当B重新开启后，它就会变成B1的从节点。

不过需要注意，如果节点B和B1同时挂了，Redis集群就无法继续正确地提供服务了。

## 3. 安装redis实例

本例使用3台服务器，每台服务器分别使用6379和7379两个端口启动2个redis实例，实现3主3从的redis集群。以下为一台服务器上的操作步骤，另两台服务器操作方法相同。

> 如果服务器资源紧张，也可使用一台服务器部署，分别使用6个不同的端口即可。

### 安装redis

本例使用的OS是Ubuntu 16.04.2 LTS，直接使用apt install安装的redis server版本就是3.0.6。

```
apt update
apt install redis-server
dpkg -l redis-server
```

### 配置redis cluster

#### 创建2个实例的配置文件

```
cd /etc/redis
mv redis.conf redis-6379.conf
cp redis-6379.conf redis-7379.conf
chown redis:redis redis-7379.conf
```

分别修改配置文件`redis-6379.conf`和`redis-7379.conf`中下列配置项：

```
daemonize yes #后台启动
port 7001 #修改端口号，从7001到7006(以实际使用为准)
pidfile /var/run/redis/redis-server-6379.pid
logfile /var/log/redis/redis-server-6379.log
dbfilename dump-6379.rdb
cluster-enabled yes #开启cluster，去掉注释
cluster-config-file nodes-6379.conf
cluster-node-timeout 15000
appendonly yes
appendfilename "appendonly-6379.aof"
```

#### 创建2个实例的服务配置文件

```
cd /lib/systemd/system
mv redis-server.service redis-server-6379.service
cp redis-server-6379.service redis-server-7379.service
```

分别修改配置文件`redis-server-6379.service`和`redis-server-7379.service`中下列配置项：

```
[Unit]
Description=Advanced key-value store
After=network.target
Documentation=http://redis.io/documentation, man:redis-server(1)

[Service]
Type=forking
ExecStart=/usr/bin/redis-server /etc/redis/redis-6379.conf
PIDFile=/var/run/redis/redis-server-6379.pid
TimeoutStopSec=0
Restart=always
User=redis
Group=redis

ExecStartPre=-/bin/run-parts --verbose /etc/redis/redis-server.pre-up.d
ExecStartPost=-/bin/run-parts --verbose /etc/redis/redis-server.post-up.d
ExecStop=-/bin/run-parts --verbose /etc/redis/redis-server.pre-down.d
ExecStop=/bin/kill -s TERM $MAINPID
ExecStopPost=-/bin/run-parts --verbose /etc/redis/redis-server.post-down.d

PrivateTmp=yes
PrivateDevices=yes
ProtectHome=yes
ReadOnlyDirectories=/
ReadWriteDirectories=-/var/lib/redis
ReadWriteDirectories=-/var/log/redis
ReadWriteDirectories=-/var/run/redis
CapabilityBoundingSet=~CAP_SYS_PTRACE

# redis-server writes its own config file when in cluster mode so we allow
# writing there (NB. ProtectSystem=true over ProtectSystem=full)
ProtectSystem=true
ReadWriteDirectories=-/etc/redis

[Install]
WantedBy=multi-user.target
Alias=redis-6379.service
```

#### 配置开机启动和启动服务

```
systemctl daemon-reload
systemctl enable redis-server-6379
systemctl enable redis-server-7379
systemctl start redis-server-6379
systemctl start redis-server-7379
```

## 4. 创建redis集群

redis提供了一个名为`redis-trib.rb`的脚本帮助我们管理redis cluster，但是要使用这个脚本，需要相应的ruby环境。

> `redis-trib.rb`脚本可以在官网提供的tar包内找到，一般在`src`目录下。

### 安装redis-trib.rb脚本所需环境

```
apt install ruby
gem install redis
```

### 使用redis-trib.rb创建集群

```
./redis-trib.rb create --replicas 1 192.168.1.1:6379 192.168.1.1:7379 192.168.1.2:6379 192.168.1.2:7379 192.168.1.3:6379 192.168.1.3:7379
```

使用create命令 --replicas 1 参数表示为每个主节点创建一个从节点，其他参数是实例的地址集合。

```
[root@localhost ~]# ./redis-trib.rb create --replicas 1 192.168.1.1:6379 192.168.1.1:7379 192.168.1.2:6379 192.168.1.2:7379 192.168.1.3:6379 192.168.1.3:7379
>>> Creating cluster  
>>> Performing hash slots allocation on 6 nodes...
Using 3 masters:
192.168.1.1:6379
192.168.1.2:6379
192.168.1.3:6379
Adding replica 192.168.1.2:7379 to 192.168.1.1:6379
Adding replica 192.168.1.3:7379 to 192.168.1.2:6379
Adding replica 192.168.1.1:7379 to 192.168.1.3:6379
M: dfd510594da614469a93a0a70767ec9145aefb1a 192.168.1.1:6379
   slots:0-5460 (5461 slots) master
M: e02eac35110bbf44c61ff90175e04d55cca097ff 192.168.1.2:6379
   slots:5461-10922 (5462 slots) master
M: 4385809e6f4952ecb122dbfedbee29109d6bb234 192.168.1.3:6379
   slots:10923-16383 (5461 slots) master
S: ec02c9ef3acee069e8849f143a492db18d4bb06c 192.168.1.2:7379
   replicates dfd510594da614469a93a0a70767ec9145aefb1a
S: 83e5a8bb94fb5aaa892cd2f6216604e03e4a6c75 192.168.1.3:7379
   replicates e02eac35110bbf44c61ff90175e04d55cca097ff
S: 10c097c429ca24f8720986c6b66f0688bfb901ee 192.168.1.1:7379
   replicates 4385809e6f4952ecb122dbfedbee29109d6bb234
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join......
>>> Performing Cluster Check (using node 192.168.1.1:6379)
M: dfd510594da614469a93a0a70767ec9145aefb1a 192.168.1.1:6379
   slots:0-5460 (5461 slots) master
M: e02eac35110bbf44c61ff90175e04d55cca097ff 192.168.1.2:6379
   slots:5461-10922 (5462 slots) master
M: 4385809e6f4952ecb122dbfedbee29109d6bb234 192.168.1.3:6379
   slots:10923-16383 (5461 slots) master
S: ec02c9ef3acee069e8849f143a492db18d4bb06c 192.168.1.2:7379
   slots: (0 slots) master
   replicates dfd510594da614469a93a0a70767ec9145aefb1a
S: 83e5a8bb94fb5aaa892cd2f6216604e03e4a6c75 192.168.1.3:7379
   slots: (0 slots) master
   replicates e02eac35110bbf44c61ff90175e04d55cca097ff
S: 10c097c429ca24f8720986c6b66f0688bfb901ee 192.168.1.1:7379
   slots: (0 slots) master
   replicates 4385809e6f4952ecb122dbfedbee29109d6bb234
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

上面显示创建成功，有3个主节点，3个从节点，每个节点都是成功连接状态。
```
:~#redis-cli -h 192.168.2.40
192.168.2.40:6379> CLUSTER info
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:3
cluster_current_epoch:6
cluster_my_epoch:5
cluster_stats_messages_sent:1610
cluster_stats_messages_received:1600
192.168.2.40:6379> CLUSTER  nodes
bf49f61dca9c3e3d2ac640ef7f0ac8b5d63237b3 192.168.2.40:6379 myself,master - 0 0 5 connected 10923-16383
b93e7c0302692ccf3a33ef9075cab0c6f88da95e 192.168.2.31:6379 master - 0 1514889538409 3 connected 5461-10922
3ae94e9f836982d0ef4b6709d3183303c4829cc5 192.168.2.21:6379 master - 0 1514889539914 1 connected 0-5460
532d01b0fade80aba5163aae144d5b3d6a00953b 192.168.2.40:7379 slave bf49f61dca9c3e3d2ac640ef7f0ac8b5d63237b3 0 1514889540916 6 connected
fa988efcfe8cabe248a3694199317ddf15274b52 192.168.2.21:7379 slave b93e7c0302692ccf3a33ef9075cab0c6f88da95e 0 1514889537909 3 connected
7c09c6836c662abd0e66c35e781c3b14b8c8d21b 192.168.2.31:7379 slave 3ae94e9f836982d0ef4b6709d3183303c4829cc5 0 1514889538911 4 connected
```
