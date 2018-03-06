# Postgres-XL集群部署与管理

## 1. 环境

* 10beta3 (Postgres-XL 10alpha2)
* CentOS Linux release 7.2.1511


## 2. Postgres-XL简介

Postgres的-XL是一个基于PostgreSQL数据库的横向扩展开源SQL数据库集群，具有足够的灵活性来处理不同的数据库工作负载:

- **完全ACID，保持事务一致性**

- **OLTP 写频繁的业务**

- **需要MPP并行性商业智能/大数据分析**

- **操作数据存储**

- **Key-value 存储**

- **GIS的地理空间**

- **混合业务工作环境**

- **多租户服务提供商托管环境**

- **Web 2.0**

  ​

  ​

  ![avatar](https://upload-images.jianshu.io/upload_images/68979-45bfbc62d3671b5f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

  ​



#### 2.1 组件简介：

- **Global Transaction Monitor (GTM)**
  全局事务管理器，确保群集范围内的事务一致性。 GTM负责发放事务ID和快照作为其多版本并发控制的一部分。
  集群可选地配置一个备用GTM，以改进可用性。此外，可以在协调器间配置代理GTM， 可用于改善可扩展性，减少GTM的通信量。
- **GTM Standby**
  GTM的备节点，在pgxc,pgxl中，GTM控制所有的全局事务分配，如果出现问题，就会导致整个集群不可用，为了增加可用性，增加该备用节点。当GTM出现问题时，GTM Standby可以升级为GTM，保证集群正常工作。
- **GTM-Proxy**
  GTM需要与所有的Coordinators通信，为了降低压力，可以在每个Coordinator机器上部署一个GTM-Proxy。
- **Coordinator**
  协调员管理用户会话，并与GTM和数据节点进行交互。协调员解析，并计划查询，并给语句中的每一个组件发送下一个序列化的全局性计划。
  为节省机器，通常此服务和数据节点部署在一起。
- **Data Node**
  数据节点是数据实际存储的地方。数据的分布可以由DBA来配置。为了提高可用性，可以配置数据节点的热备以便进行故障转移准备。

总结：gtm是负责ACID的，保证分布式数据库全局事务一致性。得益于此，就算数据节点是分布的，但是你在主节点操作增删改查事务时，就如同只操作一个数据库一样简单。Coordinator是调度的，将操作指令发送到各个数据节点。datanodes是数据节点，分布式存储数据。



## 3.  Postgres-XL环境配置与安装

准备三台Centos7服务器（或者虚拟机），集群规划如下：

|    主机名    |          IP          |       角色       |  端口   |  nodename   |       数据目录       |
| :-------: | :------------------: | :------------: | :---: | :---------: | :--------------: |
|    gtm    |  gtm.lingda.com gtm  |      GTM       | 6666  |     gtm     |    /nodes/gtm    |
|           |                      |   GTM Slave    | 20001 |  gtmSlave   | /nodes/gtmSlave  |
| datanode1 | datanode1.lingda.com |  Coordinator   | 5432  |   coord1    |   /nodes/coord   |
|           |                      |    Datanode    | 5433  |    node1    | /nodes/dn_master |
|           |                      | Datanode Slave | 15433 | node1_slave | /nodes/dn_slave  |
|           |                      |   GTM Proxy    | 6666  |  gtm_pxy1   |  /nodes/gtm_pxy  |
| datanode2 | datanode2.lingda.com |  Coordinator   | 5432  |   coord2    |   /nodes/coord   |
|           |                      |    Datanode    | 5433  |    node2    | nodes/dn_master  |
|           |                      | Datanode Slave | 15433 | node2_slave | /nodes/dn_slave  |
|           |                      |   GTM Proxy    | 6666  |  gtm_pxy2   |  /nodes/gtm_pxy  |



在每台机器的 /etc/hosts中加入以下内容：

```shell
gtm.lingda.com gtm
datanode1.lingda.com datanode1
datanode2.lingda.com datanode2
```



#### 3.1 系统环境设置

以下操作，对每个服务器节点都适用。
关闭防火墙：

```
[root@localhost ~]# systemctl stop firewalld.service
[root@localhost ~]# systemctl disable firewalld.service
```

#### 3.2 selinux设置

```
vim /etc/selinux/config

# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=disabled
# SELINUXTYPE= can take one of three two values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected. 
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted


```

安装依赖包之后重启服务器：

```
yum install -y flex bison readline-devel zlib-devel openjade docbook-style-dsssl 
```

#### 3.3 新建用户并设置免密登录

```
useradd postgres
passwd postgres
su - postgres
mkdir ~/.ssh
chmod 700 ~/.ssh

免密登录仅在gtm上执行：
su - postgres
ssh-keygen -t rsa
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

将刚生成的认证文件拷贝到databode1和datanode2中，使得gtm节点可以免密登录：

```
scp ~/.ssh/authorized_keys postgres@datanode1:~/.ssh/
scp ~/.ssh/authorized_keys postgres@datanode2:~/.ssh/
```

#### 

## 4. Postgres-XL安装

3个节点都需要安装配置，切回root用户，执行安装步骤。

#### 4.1 拉取git代码并安装

```
cd /opt
git clone git://git.postgresql.org/git/postgres-xl.git
cd postgres-xl
./configure --prefix=/home/postgres/pgxl/
make
make install
cd contrib/ 
make
make install

备注：需要提前安装gcc
```

#### 4.2 配置环境变量

```
echo $PGHOME
```

```
su - postgres
vi .bashrc
添加以下内容，并source .bashrc
export PGHOME=/home/postgres/pgxl
export LD_LIBRARY_PATH=$PGHOME/lib:$LD_LIBRARY_PATH
export PATH=$PGHOME/bin:$PATH

```

#### 4.3 集群配置

以下操作在gtm节点用postgres用户执行：

```
pgxc_ctl
PGXC prepare ---执行该命令将会生成一份配置文件模板
PGXC   ---按ctrl c退出。
```

#### 4.4 配置集群：

在pgxc_ctl文件夹中存在一个pgxc_ctl.conf文件，编辑如下：

```
#!/usr/bin/env bash

pgxcInstallDir=$HOME/pgxc
#---- OVERALL -----------------------------------------------------------------------------
#
pgxcOwner=$USER			# owner of the Postgres-XC databaseo cluster.  Here, we use this
						# both as linus user and database user.  This must be
						# the super user of each coordinator and datanode.
pgxcUser=$pgxcOwner		# OS user of Postgres-XC owner

tmpDir=/tmp					# temporary dir used in XC servers
localTmpDir=$tmpDir			# temporary dir used here locally

configBackup=n					# If you want config file backup, specify y to this value.
configBackupHost=pgxc-linker	# host to backup config file
configBackupDir=$HOME/pgxc		# Backup directory
configBackupFile=pgxc_ctl.bak	# Backup file name --> Need to synchronize when original changed.

#---- GTM ------------------------------------------------------------------------------------

# GTM is mandatory.  You must have at least (and only) one GTM master in your Postgres-XC cluster.
# If GTM crashes and you need to reconfigure it, you can do it by pgxc_update_gtm command to update
# GTM master with others.   Of course, we provide pgxc_remove_gtm command to remove it.  This command
# will not stop the current GTM.  It is up to the operator.


#---- GTM Master -----------------------------------------------

#---- Overall ----
gtmName=gtm
gtmMasterServer=gtm
gtmMasterPort=6666
gtmMasterDir=$HOME/pgxc/nodes/gtm

#---- Configuration ---
gtmExtraConfig=none			# Will be added gtm.conf for both Master and Slave (done at initilization only)
gtmMasterSpecificExtraConfig=none	# Will be added to Master's gtm.conf (done at initialization only)

#---- GTM Slave -----------------------------------------------

# Because GTM is a key component to maintain database consistency, you may want to configure GTM slave
# for backup.

#---- Overall ------

gtmSlave=y					# Specify y if you configure GTM Slave.   Otherwise, GTM slave will not be configured and
							# all the following variables will be reset.
gtmSlaveName=gtmSlave
gtmSlaveServer=gtm		# value none means GTM slave is not available.  Give none if you don't configure GTM Slave.
gtmSlavePort=20001			# Not used if you don't configure GTM slave.
gtmSlaveDir=$HOME/pgxc/nodes/gtmSlave	# Not used if you don't configure GTM slave.

# Please note that when you have GTM failover, then there will be no slave available until you configure the slave
# again. (pgxc_add_gtm_slave function will handle it)

#---- Configuration ----
gtmSlaveSpecificExtraConfig=none # Will be added to Slave's gtm.conf (done at initialization only)

#---- GTM Proxy -------------------------------------------------------------------------------------------------------


#---- Shortcuts ------
gtmProxyDir=$HOME/pgxc/nodes/gtm_pxy

#---- Overall -------
gtmProxy=y				# Specify y if you conifugre at least one GTM proxy.   You may not configure gtm proxies
						# only when you dont' configure GTM slaves.
						# If you specify this value not to y, the following parameters will be set to default empty values.
						# If we find there're no valid Proxy server names (means, every servers are specified
						# as none), then gtmProxy value will be set to "n" and all the entries will be set to
						# empty values.
gtmProxyNames=(gtm_pxy1 gtm_pxy2)	# No used if it is not configured
gtmProxyServers=(datanode1 datanode2)			# Specify none if you dont' configure it.
gtmProxyPorts=(6666 6666)				# Not used if it is not configured.
gtmProxyDirs=($gtmProxyDir $gtmProxyDir)	# Not used if it is not configured.

#---- Configuration ----
gtmPxyExtraConfig=none		# Extra configuration parameter for gtm_proxy.  Coordinator section has an example.
gtmPxySpecificExtraConfig=(none none)

#---- Coordinators ----------------------------------------------------------------------------------------------------

#---- shortcuts ----------
coordMasterDir=$HOME/pgxc/nodes/coord
coordSlaveDir=$HOME/pgxc/nodes/coord_slave
coordArchLogDir=$HOME/pgxc/nodes/coord_archlog

#---- Overall ------------
coordNames=(coord1 coord2)		# Master and slave use the same name
coordPorts=(5432 5432)			# Master ports
poolerPorts=(6667 6667)			# Master pooler ports
coordPgHbaEntries=(192.168.100.0/24)				# Assumes that all the coordinator (master/slave) accepts
												# the same connection
												# This entry allows only $pgxcOwner to connect.
												# If you'd like to setup another connection, you should
												# supply these entries through files specified below.
# Note: The above parameter is extracted as "host all all 0.0.0.0/0 trust".   If you don't want
# such setups, specify the value () to this variable and suplly what you want using coordExtraPgHba
# and/or coordSpecificExtraPgHba variables.
#coordPgHbaEntries=(::1/128)	# Same as above but for IPv6 addresses

#---- Master -------------
coordMasterServers=(datanode1 datanode2)		# none means this master is not available
coordMasterDirs=($coordMasterDir $coordMasterDir)
coordMaxWALsernder=0	# max_wal_senders: needed to configure slave. If zero value is specified,
						# it is expected to supply this parameter explicitly by external files
						# specified in the following.	If you don't configure slaves, leave this value to zero.
coordMaxWALSenders=($coordMaxWALsernder $coordMaxWALsernder)
						# max_wal_senders configuration for each coordinator.

#---- Slave -------------
coordSlave=n			# Specify y if you configure at least one coordiantor slave.  Otherwise, the following

# You can put your postgresql.conf lines here.
cat > $coordExtraConfig <<EOF
#================================================
# Added to all the coordinator postgresql.conf
# Original: $coordExtraConfig
log_destination = 'stderr'
logging_collector = on
log_directory = 'pg_log'
listen_addresses = '*'
max_connections = 100
EOF

# Additional Configuration file for specific coordinator master.
# You can define each setting by similar means as above.
coordSpecificExtraConfig=(none none)
coordExtraPgHba=none	# Extra entry for pg_hba.conf.  This file will be added to all the coordinators' pg_hba.conf
coordSpecificExtraPgHba=(none none)

#----- Additional Slaves -----
#
# Please note that this section is just a suggestion how we extend the configuration for
# multiple and cascaded replication.   They're not used in the current version.
#
coordAdditionalSlaves=n		# Additional slave can be specified as follows: where you
coordAdditionalSlaveSet=(cad1)		# Each specifies set of slaves.   This case, two set of slaves are
											# configured
cad1_Sync=n		  		# All the slaves at "cad1" are connected with asynchronous mode.
							# If not, specify "y"
							# The following lines specifies detailed configuration for each
							# slave tag, cad1.  You can define cad2 similarly.
cad1_Servers=(node08 node09 node06 node07)	# Hosts
cad1_dir=$HOME/pgxc/nodes/coord_slave_cad1
cad1_Dirs=($cad1_dir $cad1_dir $cad1_dir $cad1_dir)
cad1_ArchLogDir=$HOME/pgxc/nodes/coord_archlog_cad1
cad1_ArchLogDirs=($cad1_ArchLogDir $cad1_ArchLogDir $cad1_ArchLogDir $cad1_ArchLogDir)


#---- Datanodes -------------------------------------------------------------------------------------------------------

#---- Shortcuts --------------
datanodeMasterDir=$HOME/pgxc/nodes/dn_master
datanodeSlaveDir=$HOME/pgxc/nodes/dn_slave
datanodeArchLogDir=$HOME/pgxc/nodes/datanode_archlog

#---- Overall ---------------
#primaryDatanode=datanode1				# Primary Node.
# At present, xc has a priblem to issue ALTER NODE against the primay node.  Until it is fixed, the test will be done
# without this feature.
primaryDatanode=node1				# Primary Node.
datanodeNames=(node1 node2)
datanodePorts=(5433 5433)	# Master ports
datanodePoolerPorts=(6668 6668)	# Master pooler ports
datanodePgHbaEntries=(192.168.100.0/24)	# Assumes that all the coordinator (master/slave) accepts
										# the same connection
										# This list sets up pg_hba.conf for $pgxcOwner user.
										# If you'd like to setup other entries, supply them
										# through extra configuration files specified below.
# Note: The above parameter is extracted as "host all all 0.0.0.0/0 trust".   If you don't want
# such setups, specify the value () to this variable and suplly what you want using datanodeExtraPgHba
# and/or datanodeSpecificExtraPgHba variables.
#datanodePgHbaEntries=(::1/128)	# Same as above but for IPv6 addresses

#---- Master ----------------
datanodeMasterServers=(datanode1 datanode2)	# none means this master is not available.
													# This means that there should be the master but is down.
													# The cluster is not operational until the master is
													# recovered and ready to run.	
datanodeMasterDirs=($datanodeMasterDir $datanodeMasterDir)
datanodeMaxWalSender=4								# max_wal_senders: needed to configure slave. If zero value is 
													# specified, it is expected this parameter is explicitly supplied
													# by external configuration files.
													# If you don't configure slaves, leave this value zero.
datanodeMaxWALSenders=($datanodeMaxWalSender $datanodeMaxWalSender)
						# max_wal_senders configuration for each datanode

#---- Slave -----------------
datanodeSlave=n			# Specify y if you configure at least one coordiantor slave.  Otherwise, the following
						# configuration parameters will be set to empty values.
						# If no effective server names are found (that is, every servers are specified as none),
						# then datanodeSlave value will be set to n and all the following values will be set to
						# empty values.
datanodeSlaveServers=(datanode2 datanode1)	# value none means this slave is not available
datanodeSlavePorts=(15433 15433)	# value none means this slave is not available
datanodeSlavePoolerPorts=(20012 20012)	# value none means this slave is not available
datanodeSlaveSync=y		# If datanode slave is connected in synchronized mode
datanodeSlaveDirs=($datanodeSlaveDir $datanodeSlaveDir)
datanodeArchLogDirs=($datanodeArchLogDir $datanodeArchLogDir)

# ---- Configuration files ---
# You may supply your bash script to setup extra config lines and extra pg_hba.conf entries here.
# These files will go to corresponding files for the master.
# Or you may supply these files manually.
datanodeExtraConfig=none	# Extra configuration file for datanodes.  This file will be added to all the 
							# datanodes' postgresql.conf
datanodeSpecificExtraConfig=(none none)
datanodeExtraPgHba=none		# Extra entry for pg_hba.conf.  This file will be added to all the datanodes' postgresql.conf
datanodeSpecificExtraPgHba=(none none)

#----- Additional Slaves -----
datanodeAdditionalSlaves=n	# Additional slave can be specified as follows: where you


#---- WAL archives -------------------------------------------------------------------------------------------------
walArchive=n	# If you'd like to configure WAL archive, edit this section.
				# Pgxc_ctl assumes that if you configure WAL archive, you configure it
				# for all the coordinators and datanodes.
				# Default is "no".   Please specify "y" here to turn it on.
#
#		End of Configuration Section
#
#==========================================================================================================================

#========================================================================================================================
# The following is for extension.  Just demonstrate how to write such extension.  There's no code
# which takes care of them so please ignore the following lines.  They are simply ignored by pgxc_ctl.
# No side effects.
#=============<< Beginning of future extension demonistration >> ========================================================
# You can setup more than one backup set for various purposes, such as disaster recovery.
walArchiveSet=(war1 war2)
war1_source=(master)	# you can specify master, slave or ano other additional slaves as a source of WAL archive.
					# Default is the master
wal1_source=(slave)
wal1_source=(additiona_coordinator_slave_set additional_datanode_slave_set)
war1_host=node10	# All the nodes are backed up at the same host for a given archive set
war1_backupdir=$HOME/pgxc/backup_war1
wal2_source=(master)
war2_host=node11
war2_backupdir=$HOME/pgxc/backup_war2
#=============<< End of future extension demonistration >> ========================================================

```



## 5. 集群初始化、启动、停止、测试

#### 5.1初始化：

```shell
[postgres@gtm pgxc_ctl]$ pgxc_ctl -c /home/postgres/pgxc_ctl/pgxc_ctl.conf init all 
/bin/bash
Installing pgxc_ctl_bash script as /home/postgres/pgxc_ctl/pgxc_ctl_bash.
Installing pgxc_ctl_bash script as /home/postgres/pgxc_ctl/pgxc_ctl_bash.
Reading configuration using /home/postgres/pgxc_ctl/pgxc_ctl_bash --home /home/postgres/pgxc_ctl --configuration /home/postgres/pgxc_ctl/pgxc_ctl.conf
/home/postgres/pgxc_ctl/pgxc_ctl.conf: line 190: $coordExtraConfig: ambiguous redirect
Finished reading configuration.
ERROR: Conflict in directory names in gtmName and gtmSlaveName variable.
ERROR: Conflict in directory names in gtmSlaveName and gtmName variable.
ERROR: Found conflicts among resources.  Exiting.
[postgres@gtm pgxc_ctl]$ vim pgxc_ctl.conf 
[postgres@gtm pgxc_ctl]$ pgxc_ctl -c /home/postgres/pgxc_ctl/pgxc_ctl.conf init all 
/bin/bash
Installing pgxc_ctl_bash script as /home/postgres/pgxc_ctl/pgxc_ctl_bash.
Installing pgxc_ctl_bash script as /home/postgres/pgxc_ctl/pgxc_ctl_bash.
Reading configuration using /home/postgres/pgxc_ctl/pgxc_ctl_bash --home /home/postgres/pgxc_ctl --configuration /home/postgres/pgxc_ctl/pgxc_ctl.conf
/home/postgres/pgxc_ctl/pgxc_ctl.conf: line 190: $coordExtraConfig: ambiguous redirect
Finished reading configuration.
   ******** PGXC_CTL START ***************

Current directory: /home/postgres/pgxc_ctl
Initialize GTM master
The files belonging to this GTM system will be owned by user "postgres".
This user must also own the server process.


fixing permissions on existing directory /home/postgres/pgxc/nodes/gtm ... ok
creating configuration files ... ok
creating control file ... ok

Success.
waiting for server to shut down.... done
server stopped
Done.
Start GTM master
server starting
Initialize GTM slave
The files belonging to this GTM system will be owned by user "postgres".
This user must also own the server process.


fixing permissions on existing directory /home/postgres/pgxc/nodes/gtmSlave ... ok
creating configuration files ... ok
creating control file ... ok

Success.
Done.
Start GTM slaveserver starting
Done.
Initialize all the gtm proxies.
Initializing gtm proxy gtm_pxy1.
Initializing gtm proxy gtm_pxy2.
The files belonging to this GTM system will be owned by user "postgres".
This user must also own the server process.


fixing permissions on existing directory /home/postgres/pgxc/nodes/gtm_pxy ... ok
creating configuration files ... ok

Success.
The files belonging to this GTM system will be owned by user "postgres".
This user must also own the server process.


fixing permissions on existing directory /home/postgres/pgxc/nodes/gtm_pxy ... ok
creating configuration files ... ok

Success.
Done.
Starting all the gtm proxies.
Starting gtm proxy gtm_pxy1.
Starting gtm proxy gtm_pxy2.
server starting
server starting
Done.
Initialize all the coordinator masters.
Initialize coordinator master coord1.
Initialize coordinator master coord2.
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locale "en_US.UTF-8".
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are disabled.

fixing permissions on existing directory /home/postgres/pgxc/nodes/coord ... ok
creating subdirectories ... ok
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting dynamic shared memory implementation ... posix
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... creating cluster information ... ok
syncing data to disk ... ok
freezing database template0 ... ok
freezing database template1 ... ok
freezing database postgres ... ok

WARNING: enabling "trust" authentication for local connections
You can change this by editing pg_hba.conf or using the option -A, or
--auth-local and --auth-host, the next time you run initdb.

Success.
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locale "en_US.UTF-8".
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are disabled.

fixing permissions on existing directory /home/postgres/pgxc/nodes/coord ... ok
creating subdirectories ... ok
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting dynamic shared memory implementation ... posix
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... creating cluster information ... ok
syncing data to disk ... ok
freezing database template0 ... ok
freezing database template1 ... ok
freezing database postgres ... ok

WARNING: enabling "trust" authentication for local connections
You can change this by editing pg_hba.conf or using the option -A, or
--auth-local and --auth-host, the next time you run initdb.

Success.
Done.
Starting coordinator master.
Starting coordinator master coord1
Starting coordinator master coord2
2018-03-06 10:05:05.366 CST [31568] LOG:  listening on IPv4 address "0.0.0.0", port 5432
2018-03-06 10:05:05.366 CST [31568] LOG:  listening on IPv6 address "::", port 5432
2018-03-06 10:05:05.399 CST [31568] LOG:  listening on Unix socket "/tmp/.s.PGSQL.5432"
2018-03-06 10:05:05.458 CST [31569] LOG:  database system was shut down at 2018-03-06 10:05:02 CST
2018-03-06 10:05:05.475 CST [31568] LOG:  database system is ready to accept connections
2018-03-06 10:05:05.476 CST [31576] LOG:  cluster monitor started
2018-03-06 10:05:04.742 CST [7241] LOG:  listening on IPv4 address "0.0.0.0", port 5432
2018-03-06 10:05:04.742 CST [7241] LOG:  listening on IPv6 address "::", port 5432
2018-03-06 10:05:04.776 CST [7241] LOG:  listening on Unix socket "/tmp/.s.PGSQL.5432"
2018-03-06 10:05:04.835 CST [7242] LOG:  database system was shut down at 2018-03-06 10:05:03 CST
2018-03-06 10:05:04.852 CST [7241] LOG:  database system is ready to accept connections
2018-03-06 10:05:04.852 CST [7249] LOG:  cluster monitor started
Done.
Initialize all the datanode masters.
Initialize the datanode master node1.
Initialize the datanode master node2.
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locale "en_US.UTF-8".
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are disabled.

fixing permissions on existing directory /home/postgres/pgxc/nodes/dn_master ... ok
creating subdirectories ... ok
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting dynamic shared memory implementation ... posix
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... creating cluster information ... ok
syncing data to disk ... ok
freezing database template0 ... ok
freezing database template1 ... ok
freezing database postgres ... ok

WARNING: enabling "trust" authentication for local connections
You can change this by editing pg_hba.conf or using the option -A, or
--auth-local and --auth-host, the next time you run initdb.

Success.
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locale "en_US.UTF-8".
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are disabled.

fixing permissions on existing directory /home/postgres/pgxc/nodes/dn_master ... ok
creating subdirectories ... ok
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting dynamic shared memory implementation ... posix
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... creating cluster information ... ok
syncing data to disk ... ok
freezing database template0 ... ok
freezing database template1 ... ok
freezing database postgres ... ok

WARNING: enabling "trust" authentication for local connections
You can change this by editing pg_hba.conf or using the option -A, or
--auth-local and --auth-host, the next time you run initdb.

Success.
Done.
Starting all the datanode masters.
Starting datanode master node1.
Starting datanode master node2.
2018-03-06 10:05:25.497 CST [31911] LOG:  listening on IPv4 address "0.0.0.0", port 5433
2018-03-06 10:05:25.497 CST [31911] LOG:  listening on IPv6 address "::", port 5433
2018-03-06 10:05:25.530 CST [31911] LOG:  listening on Unix socket "/tmp/.s.PGSQL.5433"
2018-03-06 10:05:25.571 CST [31911] LOG:  redirecting log output to logging collector process
2018-03-06 10:05:25.571 CST [31911] HINT:  Future log output will appear in directory "pg_log".
2018-03-06 10:05:24.873 CST [7584] LOG:  listening on IPv4 address "0.0.0.0", port 5433
2018-03-06 10:05:24.873 CST [7584] LOG:  listening on IPv6 address "::", port 5433
2018-03-06 10:05:24.906 CST [7584] LOG:  listening on Unix socket "/tmp/.s.PGSQL.5433"
2018-03-06 10:05:24.946 CST [7584] LOG:  redirecting log output to logging collector process
2018-03-06 10:05:24.946 CST [7584] HINT:  Future log output will appear in directory "pg_log".
Done.
ALTER NODE coord1 WITH (HOST='datanode1', PORT=5432);
ALTER NODE
CREATE NODE coord2 WITH (TYPE='coordinator', HOST='datanode2', PORT=5432);
CREATE NODE
CREATE NODE node1 WITH (TYPE='datanode', HOST='datanode1', PORT=5433, PRIMARY, PREFERRED);
CREATE NODE
CREATE NODE node2 WITH (TYPE='datanode', HOST='datanode2', PORT=5433);
CREATE NODE
SELECT pgxc_pool_reload();
 pgxc_pool_reload 
------------------
 t
(1 row)

CREATE NODE coord1 WITH (TYPE='coordinator', HOST='datanode1', PORT=5432);
CREATE NODE
ALTER NODE coord2 WITH (HOST='datanode2', PORT=5432);
ALTER NODE
CREATE NODE node1 WITH (TYPE='datanode', HOST='datanode1', PORT=5433, PRIMARY);
CREATE NODE
CREATE NODE node2 WITH (TYPE='datanode', HOST='datanode2', PORT=5433, PREFERRED);
CREATE NODE
SELECT pgxc_pool_reload();
 pgxc_pool_reload 
------------------
 t
(1 row)

Done.
EXECUTE DIRECT ON (node1) 'CREATE NODE coord1 WITH (TYPE=''coordinator'', HOST=''datanode1'', PORT=5432)';
EXECUTE DIRECT
EXECUTE DIRECT ON (node1) 'CREATE NODE coord2 WITH (TYPE=''coordinator'', HOST=''datanode2'', PORT=5432)';
EXECUTE DIRECT
EXECUTE DIRECT ON (node1) 'ALTER NODE node1 WITH (TYPE=''datanode'', HOST=''datanode1'', PORT=5433, PRIMARY, PREFERRED)';
EXECUTE DIRECT
EXECUTE DIRECT ON (node1) 'CREATE NODE node2 WITH (TYPE=''datanode'', HOST=''datanode2'', PORT=5433, PREFERRED)';
EXECUTE DIRECT
EXECUTE DIRECT ON (node1) 'SELECT pgxc_pool_reload()';
 pgxc_pool_reload 
------------------
 t
(1 row)

EXECUTE DIRECT ON (node2) 'CREATE NODE coord1 WITH (TYPE=''coordinator'', HOST=''datanode1'', PORT=5432)';
EXECUTE DIRECT
EXECUTE DIRECT ON (node2) 'CREATE NODE coord2 WITH (TYPE=''coordinator'', HOST=''datanode2'', PORT=5432)';
EXECUTE DIRECT
EXECUTE DIRECT ON (node2) 'CREATE NODE node1 WITH (TYPE=''datanode'', HOST=''datanode1'', PORT=5433, PRIMARY, PREFERRED)';
EXECUTE DIRECT
EXECUTE DIRECT ON (node2) 'ALTER NODE node2 WITH (TYPE=''datanode'', HOST=''datanode2'', PORT=5433, PREFERRED)';
EXECUTE DIRECT
EXECUTE DIRECT ON (node2) 'SELECT pgxc_pool_reload()';
 pgxc_pool_reload 
------------------
 t
(1 row)

Done.
```



#### 5.2启动、停止：

```
pgxc_ctl -c /home/postgres/pgxc_ctl/pgxc_ctl.conf start all 
pgxc_ctl -c /home/postgres/pgxc_ctl/pgxc_ctl.conf stop all 
```

其他命令请从pgxc_ctl --help中获取更多信息。



#### 5.3测试

##### 5.3.1插入数据

在datanode1进入数据库

```
psql -p 5432
select * from pgxc_node;

 node_name | node_type | node_port | node_host | nodeis_primary | nodeis_preferred |   node_id   
-----------+-----------+-----------+-----------+----------------+------------------+-------------
 coord1    | C         |      5432 | datanode1 | f              | f                |  1885696643
 coord2    | C         |      5432 | datanode2 | f              | f                | -1197102633
 node1     | D         |      5433 | datanode1 | t              | t                |  1148549230
 node2     | D         |      5433 | datanode2 | f              | f                |  -927910690


create table test1(id int,name text);
insert into test1(id,name) select generate_series(1,8),'测试';

```

##### 5.3.2查看数据分布

分别在datanode1和datanode2上查看数据（5433端口）

```
psql -p 5433
select * from test1;
```



## 6. 集群应用与管理

#### 6.1 建表说明

- REPLICATION表：各个datanode节点中，表的数据完全相同，也就是说，插入数据时，会分别在每个datanode节点插入相同数据。读数据时，只需要读任意一个datanode节点上的数据。
  建表语法：

```
postgres=#  CREATE TABLE repltab (col1 int, col2 int) DISTRIBUTE BY REPLICATION;

```

- DISTRIBUTE ：会将插入的数据，按照拆分规则，分配到不同的datanode节点中存储，也就是sharding技术。每个datanode节点只保存了部分数据，通过coordinate节点可以查询完整的数据视图。

```
postgres=#  CREATE TABLE disttab(col1 int, col2 int, col3 text) DISTRIBUTE BY HASH(col1);
```

模拟部分数据，插入测试数据：

```
#任意登录一个coordinate节点进行建表操作
[postgres@gtm ~]$  psql -h  datanode1 -p 5432 -U postgres
postgres=# INSERT INTO disttab SELECT generate_series(1,100), generate_series(101, 200), 'foo';
INSERT 0 100
postgres=# INSERT INTO repltab SELECT generate_series(1,100), generate_series(101, 200);
INSERT 0 100

```

查看数据分布结果：

```
#DISTRIBUTE表分布结果
postgres=# SELECT xc_node_id, count(*) FROM disttab GROUP BY xc_node_id;
 xc_node_id | count 
------------+-------
 1148549230 |    42
 -927910690 |    58
(2 rows)
#REPLICATION表分布结果
postgres=# SELECT count(*) FROM repltab;
 count 
-------
   100
(1 row)


```

查看另一个datanode2中repltab表结果

```
[postgres@datanode2 pgxl9.5]$ psql -p 5433
psql (PGXL 9.5r1.3, based on PG 9.5.4 (Postgres-XL 9.5r1.3))
Type "help" for help.

postgres=# SELECT count(*) FROM repltab;
 count 
-------
   100
(1 row)


```

结论：REPLICATION表中，datanode1,datanode2中表是全部数据，一模一样。而DISTRIBUTE表，数据散落近乎平均分配到了datanode1,datanode2节点中。

#### 6.2新增datanode节点与数据重分布

##### 6.2.1 新增datanode节点

在gtm集群管理节点上执行pgxc_ctl命令

```
[postgres@gtm ~]$ pgxc_ctl
/bin/bash
Installing pgxc_ctl_bash script as /home/postgres/pgxc_ctl/pgxc_ctl_bash.
Installing pgxc_ctl_bash script as /home/postgres/pgxc_ctl/pgxc_ctl_bash.
Reading configuration using /home/postgres/pgxc_ctl/pgxc_ctl_bash --home /home/postgres/pgxc_ctl --configuration /home/postgres/pgxc_ctl/pgxc_ctl.conf
Finished reading configuration.
   ******** PGXC_CTL START ***************

Current directory: /home/postgres/pgxc_ctl
PGXC 


```

在PGXC后面执行新增数据节点命令:

```
Current directory: /home/postgres/pgxc_ctl
# 在服务器datanode1上，新增一个master角色的datanode节点，名称是dn3
# 端口号暂定5430，pool master暂定6669 ，指定好数据目录位置，从两个节点升级到3个节点，之后要写3个none
# none应该是datanodeSpecificExtraConfig或者datanodeSpecificExtraPgHba配置
PGXC add datanode master dn3 datanode1 5430 6669 /home/postgres/pgxl9.5/data/nodes/dn_master3 none none none

```

等待新增完成后，查询集群节点状态：

```
[postgres@gtm ~]$ psql -h datanode1 -p 5432 -U postgres
psql (PGXL 9.5r1.3, based on PG 9.5.4 (Postgres-XL 9.5r1.3))
Type "help" for help.

postgres=# select * from pgxc_node;
 node_name | node_type | node_port | node_host | nodeis_primary | nodeis_preferred |   node_id   
-----------+-----------+-----------+-----------+----------------+------------------+-------------
 coord1    | C         |      5432 | datanode1 | f              | f                |  1885696643
 coord2    | C         |      5432 | datanode2 | f              | f                | -1197102633
 node1     | D         |      5433 | datanode1 | f              | t                |  1148549230
 node2     | D         |      5433 | datanode2 | f              | f                |  -927910690
 dn3       | D         |      5430 | datanode1 | f              | f                |  -700122826
(5 rows)


```

可以发现节点新增完毕。

##### 6.2.2 数据重分布

之前我们的DISTRIBUTE表分布在了node1,node2节点上，如下：

```
postgres=# SELECT xc_node_id, count(*) FROM disttab GROUP BY xc_node_id;
 xc_node_id | count 
------------+-------
 1148549230 |    42
 -927910690 |    58
(2 rows)

```

新增一个节点后，将sharding表数据重新分配到三个节点上，将repl表复制到新节点：

```
# 重分布sharding表
postgres=# ALTER TABLE disttab ADD NODE (dn3);
ALTER TABLE
# 复制数据到新节点
postgres=#  ALTER TABLE repltab ADD NODE (dn3);
ALTER TABLE


```

查看新的数据分布：

```
postgres=# SELECT xc_node_id, count(*) FROM disttab GROUP BY xc_node_id;
 xc_node_id | count 
------------+-------
 -700122826 |    36
 -927910690 |    32
 1148549230 |    32
(3 rows)


```

登录dn3(新增的时候，放在了datanode1服务器上，端口5430)节点查看数据：

```
[postgres@gtm ~]$ psql -h datanode1 -p 5430 -U postgres
psql (PGXL 9.5r1.3, based on PG 9.5.4 (Postgres-XL 9.5r1.3))
Type "help" for help.
postgres=# select count(*) from repltab;
 count 
-------
   100
(1 row)


```

很明显,通过 ALTER TABLE tt ADD NODE (dn)命令，可以将DISTRIBUTE表数据重新分布到新节点，重分布过程中会中断所有事务。可以将REPLICATION表数据复制到新节点。

##### 6.2.3 从datanode节点中回收数据

```
postgres=# ALTER TABLE disttab DELETE NODE (dn3);
ALTER TABLE
postgres=# ALTER TABLE repltab DELETE NODE (dn3);
ALTER TABLE

```

#### 6.3 删除数据节点

Postgresql-XL并没有检查将被删除的datanode节点是否有replicated/distributed表的数据，为了数据安全，在删除之前需要检查下被删除节点上的数据，有数据的话，要回收掉分配到其他节点，然后才能安全删除。删除数据节点分为四步骤：

- 查询要删除节点dn3的oid

```
postgres=#  SELECT oid, * FROM pgxc_node;
  oid  | node_name | node_type | node_port | node_host | nodeis_primary | nodeis_preferred |   node_id   
-------+-----------+-----------+-----------+-----------+----------------+------------------+-------------
 11819 | coord1    | C         |      5432 | datanode1 | f              | f                |  1885696643
 16384 | coord2    | C         |      5432 | datanode2 | f              | f                | -1197102633
 16385 | node1     | D         |      5433 | datanode1 | f              | t                |  1148549230
 16386 | node2     | D         |      5433 | datanode2 | f              | f                |  -927910690
 16397 | dn3       | D         |      5430 | datanode1 | f              | f                |  -700122826
(5 rows)
```

- 查询dn3对应的oid中是否有数据

```
testdb=# SELECT * FROM pgxc_class WHERE nodeoids::integer[] @> ARRAY[16397];
 pcrelid | pclocatortype | pcattnum | pchashalgorithm | pchashbuckets |     nodeoids      
---------+---------------+----------+-----------------+---------------+-------------------
   16388 | H             |        1 |               1 |          4096 | 16397 16385 16386
   16394 | R             |        0 |               0 |             0 | 16397 16385 16386
(2 rows)

```

- 有数据的先回收数据

```
postgres=# ALTER TABLE disttab DELETE NODE (dn3);
ALTER TABLE
postgres=# ALTER TABLE repltab DELETE NODE (dn3);
ALTER TABLE
postgres=# SELECT * FROM pgxc_class WHERE nodeoids::integer[] @> ARRAY[16397];
 pcrelid | pclocatortype | pcattnum | pchashalgorithm | pchashbuckets | nodeoids 
---------+---------------+----------+-----------------+---------------+----------
(0 rows)

```

- 安全删除dn3

```
PGXC$  remove datanode master dn3 clean

```

#### 6.4 coordinate节点管理

同datanode节点相似，列出语句不做测试了：

```
# 新增coordinate
PGXC$  add coordinator master coord3 localhost 30003 30013 $dataDirRoot/coord_master.3 none none none
# 删除coordinate,clean选项可以将相应的数据目录也删除
PGXC$  remove coordinator master coord3 clean

```

#### 6.5 故障切换

- 查看当前数据集群

```
postgres=# SELECT oid, * FROM pgxc_node;
  oid  | node_name | node_type | node_port | node_host | nodeis_primary | nodeis_preferred |   node_id   
-------+-----------+-----------+-----------+-----------+----------------+------------------+-------------
 11819 | coord1    | C         |      5432 | datanode1 | f              | f                |  1885696643
 16384 | coord2    | C         |      5432 | datanode2 | f              | f                | -1197102633
 16385 | node1     | D         |      5433 | datanode1 | f              | t                |  1148549230
 16386 | node2     | D         |      5433 | datanode2 | f              | f                |  -927910690
(4 rows)
```

- 模拟node1节点故障

```
PGXC$  stop -m immediate datanode master node1
Stopping datanode master node1.
Done.

```

- 测试集群查询

```
postgres=# SELECT xc_node_id, count(*) FROM disttab GROUP BY xc_node_id;
ERROR:  Failed to get pooled connections
postgres=# SELECT xc_node_id, * FROM disttab WHERE col1 = 3;
 xc_node_id | col1 | col2 | col3 
------------+------+------+------
 -927910690 |    3 |  103 | foo
(1 row)

```

测试发现，查询范围如果涉及到故障的node1节点，会报错，而查询的数据范围不在node1上的话，仍然可以查询。

- 手动切换node1的slave

```
PGXC$  failover datanode node1
# 切换完成后，查询集群
postgres=# SELECT oid, * FROM pgxc_node;
  oid  | node_name | node_type | node_port | node_host | nodeis_primary | nodeis_preferred |   node_id   
-------+-----------+-----------+-----------+-----------+----------------+------------------+-------------
 11819 | coord1    | C         |      5432 | datanode1 | f              | f                |  1885696643
 16384 | coord2    | C         |      5432 | datanode2 | f              | f                | -1197102633
 16386 | node2     | D         |      5433 | datanode2 | f              | f                |  -927910690
 16385 | node1     | D         |     15433 | datanode2 | f              | t                |  1148549230
(4 rows)
```

发现node1节点的ip和端口都已经替换为配置的slave了。