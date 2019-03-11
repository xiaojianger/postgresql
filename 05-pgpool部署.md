## 0 环境说明

主机名 | 组件 | IP地址 | 端口 | 版本 | VIP
----|---|---|--------|---|---
c6 | 主库 | 10.204.23.82 | 5432/9999 | pg10.1/pgpool-II 3.6.6 | 10.204.23.88 
c61 | 备库 | 10.204.23.83 | 5432/9999 | pg10.1/pgpool-II 3.6.6 | 10.204.23.88 


## 1 主从库编译安装
```
tar -xvf /opt/pgpool-II-3.6.14.tar.gz
cd /opt/pgpool-II-3.6.14
./configure --prefix=/opt/pgpool --with-pgsql=/usr/local/pgsql
make
make install
ln -s /opt/pgpool /usr/local/pgpool
echo 'export PATH=/usr/local/pgpool/bin:$PATH' >> /etc/profile && source /etc/profile
echo 'export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/pgpool/lib' >> /etc/profile && source /etc/profile
```
## 2 主从OS系统配置
```
vi /etc/hosts
10.204.23.82    c6
10.204.23.83    c62

#配置root与postgres用户互信
#主c6
# ssh-keygen
# ssh-copy-id postgres@c62
#从c62
# ssh-keygen
# ssh-copy-id postgres@c6

```
## 3 认证配置文件设置
```
cd /usr/local/pgpool/etc/
cp pool_hba.conf.sample pool_hba.conf
vi pool_hba.conf
host    replication repuser 10.204.23.82/32 md5
host    replication repuser 10.204.23.83/32 md5
host    replication repuser 10.204.23.88/32 md5
host    all all 0.0.0.0/0 md5

cp pgpool.conf.sample-stream pgpool.conf
pg_md5 -u postgres -m 123456
pg_md5 -u repuser -m 123456 
```
## 4 主库pgpool.conf
```
listen_addresses = '*'
port = 9999
socket_dir = '/tmp'
listen_backlog_multiplier = 2
serialize_accept = off

# - pgpool Communication Manager Connection Settings -
pcp_listen_addresses = '*'
pcp_port = 9898
pcp_socket_dir = '/tmp'

# - Backend Connection Settings -
backend_hostname0 = 'c6'
backend_port0 = 5432
backend_weight0 = 1
backend_data_directory0 = '/database/pgl0/pg_root'
backend_flag0 = 'ALLOW_TO_FAILOVER'
backend_hostname1 = 'c62'
backend_port1 = 5432
backend_weight1 = 1
backend_data_directory1 = '/database/pgl0/pg_root'
backend_flag1 = 'ALLOW_TO_FAILOVER'

# - Authentication -
enable_pool_hba = on
pool_passwd = 'pool_passwd'
authentication_timeout = 60

# - Concurrent session and pool size -
num_init_children = 32
max_pool = 4

# - Life time -
child_life_time = 300
child_max_connections = 0
connection_life_time = 0
client_idle_limit = 0

# - Where to log -
log_destination = 'syslog'

# - What to log -
log_line_prefix = '%t: pid %p: '
log_hostname = on
log_statement = on
log_per_node_statement = on
log_standby_delay = 'none'

# - Syslog specific -
syslog_facility = 'LOCAL0'
syslog_ident = 'pgpool'


pid_file_name = '/usr/local/pgpool/pgpool.pid'
logdir = '/usr/local/pgpool/log'

connection_cache = on
reset_query_list = 'ABORT; DISCARD ALL'
replication_mode = off
replicate_select = off
insert_lock = on
lobj_lock_table = ''

# - Degenerate handling -
replication_stop_on_mismatch = off
failover_if_affected_tuples_mismatch = off

# LOAD BALANCING MODE
load_balance_mode = off
ignore_leading_white_space = on
white_function_list = ''
black_function_list = 'nextval,setval,nextval,setval'
database_redirect_preference_list = ''
app_name_redirect_preference_list = ''
allow_sql_comments = off

# MASTER/SLAVE MODE
master_slave_mode = on
master_slave_sub_mode = 'stream'

# - Streaming -
sr_check_period = 10
sr_check_user = 'repuser'
sr_check_password = '123456'
sr_check_database = 'postgres'
delay_threshold = 10000000

# - Special commands -
follow_master_command = ''

# HEALTH CHECK
health_check_period = 5
health_check_timeout = 20
health_check_user = 'repuser'
health_check_password = '123456'
health_check_database = 'postgres'
health_check_max_retries = 3
health_check_retry_delay = 3
connect_timeout = 10000

# FAILOVER AND FAILBACK
failover_command = '/usr/local/pgpool/etc/failover_stream.sh %d %P %H %R'
failback_command = ''
fail_over_on_backend_error = on
search_primary_node_timeout = 300

# ONLINE RECOVERY
recovery_user = 'nobody'
recovery_password = ''
recovery_1st_stage_command = ''
recovery_2nd_stage_command = ''
recovery_timeout = 90
client_idle_limit_in_recovery = 0

# WATCHDOG
# - Enabling -
use_watchdog = on

# -Connection to up stream servers -
trusted_servers = ''
ping_path = '/bin'

# - Watchdog communication Settings -
wd_hostname = 'c6'
wd_port = 9000
wd_priority = 1
wd_authkey = ''
wd_ipc_socket_dir = '/tmp'

# - Virtual IP control Setting -
delegate_IP = '10.204.23.88'
if_cmd_path = '/sbin'
if_up_cmd = 'ip addr add $_IP_$/24 dev eth0 label eth0:0'
if_down_cmd = 'ip addr del $_IP_$/24 dev eth0'
arping_path = '/usr/sbin'
arping_cmd = 'arping -U $_IP_$ -w 1'

# - Behaivor on escalation Setting -
clear_memqcache_on_escalation = on
wd_escalation_command = ''
wd_de_escalation_command = ''

# - Lifecheck Setting -
wd_monitoring_interfaces_list = '' 
wd_lifecheck_method = 'heartbeat'
wd_interval = 10

# -- heartbeat mode --
wd_heartbeat_port = 9694
wd_heartbeat_keepalive = 2
wd_heartbeat_deadtime = 30
heartbeat_destination0 = 'c62'
heartbeat_destination_port0 = 9694 
heartbeat_device0 = 'eth0'

#heartbeat_destination1 = 'host0_ip2'
#heartbeat_destination_port1 = 9694
#heartbeat_device1 = ''

# -- query mode --

wd_life_point = 3
wd_lifecheck_query = 'SELECT 1'
wd_lifecheck_dbname = 'postgres'
wd_lifecheck_user = 'repuser'
wd_lifecheck_password = '123456'

# - Other pgpool Connection Settings -

other_pgpool_hostname0 = 'c62'
other_pgpool_port0 = 9999
other_wd_port0 = 9000
#other_pgpool_hostname1 = 'host1'
#other_pgpool_port1 = 5432
#other_wd_port1 = 9000

# OTHERS
relcache_expire = 0
relcache_size = 256
check_temp_table = on
check_unlogged_table = on

# IN MEMORY QUERY MEMORY CACHE
memory_cache_enabled = off
memqcache_method = 'shmem'
memqcache_memcached_host = 'localhost'
memqcache_memcached_port = 11211
memqcache_total_size = 67108864
memqcache_max_num_cache = 1000000
memqcache_expire = 0
memqcache_auto_cache_invalidation = on
memqcache_maxcache = 409600
memqcache_cache_block_size = 1048576
memqcache_oiddir = '/var/log/pgpool/oiddir'
white_memqcache_table_list = ''
```
## 5 备库pgpool.conf
```
listen_addresses = '*'
port = 9999
socket_dir = '/tmp'
listen_backlog_multiplier = 2
serialize_accept = off

# - pgpool Communication Manager Connection Settings -
pcp_listen_addresses = '*'
pcp_port = 9898
pcp_socket_dir = '/tmp'

# - Backend Connection Settings -
backend_hostname0 = 'c6'
backend_port0 = 5432
backend_weight0 = 1
backend_data_directory0 = '/database/pgl0/pg_root'
backend_flag0 = 'ALLOW_TO_FAILOVER'
backend_hostname1 = 'c62'
backend_port1 = 5432
backend_weight1 = 1
backend_data_directory1 = '/database/pgl0/pg_root'
backend_flag1 = 'ALLOW_TO_FAILOVER'

# - Authentication -
enable_pool_hba = on
pool_passwd = 'pool_passwd'
authentication_timeout = 60

# - Concurrent session and pool size -
num_init_children = 32
max_pool = 4

# - Life time -
child_life_time = 300
child_max_connections = 0
connection_life_time = 0
client_idle_limit = 0

# - Where to log -
log_destination = 'syslog'

# - What to log -

log_line_prefix = '%t: pid %p: '
log_connections = on
log_hostname = on
log_statement = on
log_per_node_statement = on
log_standby_delay = 'none'

# - Syslog specific -
syslog_facility = 'LOCAL0'
syslog_ident = 'pgpool'
pid_file_name = '/usr/local/pgpool/pgpool.pid'
logdir = '/usr/local/pgpool/log'
connection_cache = on
reset_query_list = 'ABORT; DISCARD ALL'
replication_mode = off
replicate_select = off
insert_lock = on
lobj_lock_table = ''

# - Degenerate handling -
replication_stop_on_mismatch = off
failover_if_affected_tuples_mismatch = off


# LOAD BALANCING MODE
load_balance_mode = off
ignore_leading_white_space = on
white_function_list = ''
black_function_list = 'nextval,setval,nextval,setval'
database_redirect_preference_list = ''
app_name_redirect_preference_list = ''
allow_sql_comments = off

# MASTER/SLAVE MODE
master_slave_mode = on
master_slave_sub_mode = 'stream'

# - Streaming -
sr_check_period = 10
sr_check_user = 'repuser'
sr_check_password = '123456'
sr_check_database = 'postgres'
delay_threshold = 10000000

# - Special commands -
follow_master_command = ''

# HEALTH CHECK
health_check_period = 5
health_check_timeout = 20
health_check_user = 'repuser'
health_check_password = '123456'
health_check_database = 'postgres'
health_check_max_retries = 3
health_check_retry_delay = 3
connect_timeout = 10000

# FAILOVER AND FAILBACK
failover_command = '/usr/local/pgpool/etc/failover_stream.sh %d %P %H %R'
failback_command = ''
fail_over_on_backend_error = on
search_primary_node_timeout = 300

# ONLINE RECOVERY
recovery_user = 'nobody'
recovery_password = ''
recovery_1st_stage_command = ''
recovery_2nd_stage_command = ''
recovery_timeout = 90
client_idle_limit_in_recovery = 0

# WATCHDOG
use_watchdog = on

# -Connection to up stream servers -
trusted_servers = ''
ping_path = '/bin'

# - Watchdog communication Settings -
wd_hostname = 'c62'
wd_port = 9000
wd_priority = 1
wd_authkey = ''
wd_ipc_socket_dir = '/tmp'


# - Virtual IP control Setting -
delegate_IP = '10.204.23.88'
if_cmd_path = '/sbin'
if_up_cmd = 'ip addr add $_IP_$/24 dev eth0 label eth0:0'
if_down_cmd = 'ip addr del $_IP_$/24 dev eth0'
arping_path = '/usr/sbin'
arping_cmd = 'arping -U $_IP_$ -w 1'

# - Behaivor on escalation Setting -
clear_memqcache_on_escalation = on
wd_escalation_command = ''
wd_de_escalation_command = ''

# - Lifecheck Setting -
wd_monitoring_interfaces_list = ''
wd_lifecheck_method = 'heartbeat'
wd_interval = 10

# -- heartbeat mode --
wd_heartbeat_port = 9694
wd_heartbeat_keepalive = 2
wd_heartbeat_deadtime = 30
heartbeat_destination0 = 'c6'
heartbeat_destination_port0 = 9694
heartbeat_device0 = 'eth0'


# -- query mode --
wd_life_point = 3
wd_lifecheck_query = 'SELECT 1'
wd_lifecheck_dbname = 'postgres'
wd_lifecheck_user = 'repuser'
wd_lifecheck_password = '123456'

# - Other pgpool Connection Settings -
other_pgpool_hostname0 = 'c6'
other_pgpool_port0 = 9999
other_wd_port0 = 9000

# OTHERS
relcache_expire = 0
relcache_size = 256
check_temp_table = on
check_unlogged_table = on

# IN MEMORY QUERY MEMORY CACHE
memory_cache_enabled = off
memqcache_method = 'shmem'
memqcache_memcached_host = 'localhost'
memqcache_memcached_port = 11211
memqcache_total_size = 67108864
memqcache_max_num_cache = 1000000
memqcache_expire = 0
memqcache_auto_cache_invalidation = on
memqcache_maxcache = 409600
memqcache_cache_block_size = 1048576
memqcache_oiddir = '/var/log/pgpool/oiddir'
white_memqcache_table_list = ''
black_memqcache_table_list = ''
```
## 6 故障切换脚本
配置文件中已配置位置：failover_command = '/usr/local/pgpool/etc/failover_stream.sh %d %P %H %R'
```
#! /bin/bash
# Execute command by failover.
# special values: %d = node id
#                 %h = host name
#                 %p = port number
#                 %D = database cluster path
#                 %m = new master node id
#                 %M = old master node id
#                 %H = new master node host name
#                 %P = old primary node id
#                 %R = new master database cluster path
#                 %r = new master port number
#                 %% = '%' character

falling_node=$1   # %d
old_primary=$2    # %P
new_primary=$3    # %H
pgdata=$4         # %R
pghome=/usr/local/pgsql
log=/tmp/failover.log
date >> $log

echo "falling_node=$falling_node" >> $log
echo "old_primary=$old_primary" >> $log
echo "new_primary=$new_primary" >> $log
echo "pgdata=$pgdata" >> $log

if [ $falling_node = $old_primary ] && [ $UID -eq 0 ]; then

   if [ -f $pgdata/recovery.conf ]; then
      su postgres -c "$pghome/bin/pg_ctl promote -D $pgdata"
      echo "Local promote" >> $log
   else
      su postgres -c "ssh -T postgres@$new_primary $pghome/bin/pg_ctl promote -D $pgdata"
      echo "Remote promote" >> $log
   fi
fi
exit 0;   
```
## 7 启停pgpool
```
#启动
[root@c6 ~]# pgpool
#停止
[root@c6 ~]# pgpool -m fast stop
```
## 8 使用pgpool连接数据库
```
[postgres@c62 ~]$ psql -Urepuser -h10.204.23.88 -p9999  mytest
Password for user repuser: 
psql (10.7)
Type "help" for help.
```
## 9 配置PCP管理
```
#创建一个pgpool管理用户（和连接pg实例无关系）
#pgpool所有节点都需要配置，方便后续在任意一个节点上都可以操作管理
[postgres@c6 ~]$ pg_md5 pgpool
ba777e4c2f15c11ea8ac3be7e0440aa0
[postgres@c6 ~]$ cp /usr/local/pgpool/etc/pcp.conf.sample /usr/local/pgpool/etc/pcp.conf
[postgres@c6 ~]$ vi /usr/local/pgpool/etc/pcp.conf
pgpool:ba777e4c2f15c11ea8ac3be7e0440aa0
```
## 10 pgpool常用命令
```
#查看pgpool的节点
mytest=> show pool_nodes;
 node_id | hostname | port | status | lb_weight |  role   | select_cnt | load_balance_node | replication_delay 
---------+----------+------+--------+-----------+---------+------------+-------------------+-------------------
 0       | c6       | 5432 | up     | 0.500000  | primary | 0          | true              | 0
 1       | c62      | 5432 | up     | 0.500000  | standby | 0          | false             | 0
(2 rows)

#使用PCP查看各个节点的状态
[root@c6 ~]# pcp_watchdog_info --verbose -h 10.204.23.82 -U  pgpool
Password: 
Watchdog Cluster Information 
Total Nodes          : 2
Remote Nodes         : 1
Quorum state         : QUORUM EXIST
Alive Remote Nodes   : 1
VIP up on local node : YES
Master Node Name     : c6:9999 Linux c6
Master Host Name     : c6

Watchdog Node Information 
Node Name      : c6:9999 Linux c6
Host Name      : c6
Delegate IP    : 10.204.23.88
Pgpool port    : 9999
Watchdog port  : 9000
Node priority  : 1
Status         : 4
Status Name    : MASTER

Node Name      : c62:9999 Linux c62
Host Name      : c62
Delegate IP    : 10.204.23.88
Pgpool port    : 9999
Watchdog port  : 9000
Node priority  : 1
Status         : 7
Status Name    : STANDBY
##pg数据库关闭后，再加入pgpool中需要手动添加
pcp_attach_node -h 10.204.23.88 -U pgpool 0
### 0是pool_nodes列表中需要

##故障切换时
使用pg_ctl promote  [-D DATADIR] [-W] [-t SECS] [-s] 将从库生成主库

##将数据库恢复成从库
将recover.done 文件 mv 成recover.conf，再重启数据库
```
## 11 故障切换测试
#### 11.1 测试pgpool程序的高可用，关闭pgpool主节点时，测试是否实现故障转移
```
##主上查看当前状态（VIP绑定在主节点）
[root@c6 ~]# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 00:50:56:9f:40:c9 brd ff:ff:ff:ff:ff:ff
    inet 10.204.23.82/24 brd 10.204.23.255 scope global eth0
    inet 10.204.23.88/24 scope global secondary eth0:0
    inet6 fe80::250:56ff:fe9f:40c9/64 scope link 
       valid_lft forever preferred_lft forever
##关闭主节点pgpool
[root@c6 ~]# pgpool -m fast stop
.done.
##主上查看当前状态（VIP已解绑）
[root@c6 ~]# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 00:50:56:9f:40:c9 brd ff:ff:ff:ff:ff:ff
    inet 10.204.23.82/24 brd 10.204.23.255 scope global eth0
    inet6 fe80::250:56ff:fe9f:40c9/64 scope link 
       valid_lft forever preferred_lft forever
##从上查看状态（VIP已绑定）
[root@c62 etc]# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 00:50:56:9f:2f:56 brd ff:ff:ff:ff:ff:ff
    inet 10.204.23.83/24 brd 10.204.23.255 scope global eth0
    inet 10.204.23.88/24 scope global secondary eth0:0
    inet6 fe80::250:56ff:fe9f:2f56/64 scope link 
       valid_lft forever preferred_lft forever
##远程访问VIP地址（可以创建连接）
[root@testmanager ~]# su - postgres
-bash-4.1$ 
-bash-4.1$ psql -h10.204.23.88 -U postgres 
Password for user postgres: 
psql (8.4.20, server 10.7)
WARNING: psql version 8.4, server version 10.7.
         Some psql features might not work.
Type "help" for help.

postgres=# 

```
#### 11.2关闭主数据库，测试是否能实现故障转移
```
##恢复环境到初始状态
[root@c6 log]# pcp_watchdog_info --verbose -h 10.204.23.82 -U  pgpool 
Password: 
Watchdog Cluster Information 
Total Nodes          : 2
Remote Nodes         : 1
Quorum state         : QUORUM EXIST
Alive Remote Nodes   : 1
VIP up on local node : YES
Master Node Name     : c6:9999 Linux c6
Master Host Name     : c6

Watchdog Node Information 
Node Name      : c6:9999 Linux c6
Host Name      : c6
Delegate IP    : 10.204.23.88
Pgpool port    : 9999
Watchdog port  : 9000
Node priority  : 1
Status         : 4
Status Name    : MASTER

Node Name      : c62:9999 Linux c62
Host Name      : c62
Delegate IP    : 10.204.23.88
Pgpool port    : 9999
Watchdog port  : 9000
Node priority  : 1
Status         : 7
Status Name    : STANDBY
##查看pg后端节点信息
-bash-4.1$ psql -h10.204.23.88 -U postgres -p 9999
Password for user postgres: 
psql (8.4.20, server 10.7)
WARNING: psql version 8.4, server version 10.7.
         Some psql features might not work.
Type "help" for help.

postgres=# show pool_nodes;
 node_id | hostname | port | status | lb_weight |  role   | select_cnt | load_balance_node | replication_delay 
---------+----------+------+--------+-----------+---------+------------+-------------------+-------------------
 0       | c6       | 5432 | up     | 0.500000  | primary | 0          | true              | 0
 1       | c62      | 5432 | up     | 0.500000  | standby | 6          | false             | 0
(2 rows)

##查看备数据库的状态
[postgres@c62 ~]$ pg_controldata | grep cluster
Database cluster state:               in archive recovery
#备库处于应用归档状态，无法写入

##关闭主数据库
[postgres@c6 ~]$ pg_ctl -m fast stop
waiting for server to shut down.... done
server stopped

##查看备数据库的状态
[postgres@c62 ~]$ pg_controldata | grep cluster
Database cluster state:               in production
#可以看出备库已变为生产使用状态

##查看pg后端节点信息
-bash-4.1$ psql -h10.204.23.88 -U postgres -p 9999
Password for user postgres: 
psql (8.4.20, server 10.7)
WARNING: psql version 8.4, server version 10.7.
         Some psql features might not work.
Type "help" for help.

postgres=# show pool_nodes;
 node_id | hostname | port | status | lb_weight |  role   | select_cnt | load_balance_node | replication_delay 
---------+----------+------+--------+-----------+---------+------------+-------------------+-------------------
 0       | c6       | 5432 | down   | 0.500000  | standby | 0          | false             | 0
 1       | c62      | 5432 | up     | 0.500000  | primary | 0          | true              | 0
#可以看出主库已挂，从库变成主库
```
#### 11.3关闭主库主机，测试是否实现故障转移
```
##恢复环境到初始状态
[root@c6 log]# pcp_watchdog_info --verbose -h 10.204.23.82 -U  pgpool 
Password: 
Watchdog Cluster Information 
Total Nodes          : 2
Remote Nodes         : 1
Quorum state         : QUORUM EXIST
Alive Remote Nodes   : 1
VIP up on local node : YES
Master Node Name     : c6:9999 Linux c6
Master Host Name     : c6

Watchdog Node Information 
Node Name      : c6:9999 Linux c6
Host Name      : c6
Delegate IP    : 10.204.23.88
Pgpool port    : 9999
Watchdog port  : 9000
Node priority  : 1
Status         : 4
Status Name    : MASTER

Node Name      : c62:9999 Linux c62
Host Name      : c62
Delegate IP    : 10.204.23.88
Pgpool port    : 9999
Watchdog port  : 9000
Node priority  : 1
Status         : 7
Status Name    : STANDBY
##查看pg后端节点信息
-bash-4.1$ psql -h10.204.23.88 -U postgres -p 9999
Password for user postgres: 
psql (8.4.20, server 10.7)
WARNING: psql version 8.4, server version 10.7.
         Some psql features might not work.
Type "help" for help.

postgres=# show pool_nodes;
 node_id | hostname | port | status | lb_weight |  role   | select_cnt | load_balance_node | replication_delay 
---------+----------+------+--------+-----------+---------+------------+-------------------+-------------------
 0       | c6       | 5432 | up     | 0.500000  | primary | 0          | true              | 0
 1       | c62      | 5432 | up     | 0.500000  | standby | 6          | false             | 0
(2 rows)

##查看备数据库的状态
[postgres@c62 ~]$ pg_controldata | grep cluster
Database cluster state:               in archive recovery
#备库处于应用归档状态，无法写入

##关闭主机
[root@c6 etc]# reboot

##查看备数据库的状态
[postgres@c62 pg_root]$ pg_controldata | grep cluster
Database cluster state:               in production
#可以看出备库已变为生产使用状态

##查看pg后端节点信息
postgres=# show pool_nodes;
 node_id | hostname | port | status | lb_weight |  role   | select_cnt | load_balance_node | replication_delay 
---------+----------+------+--------+-----------+---------+------------+-------------------+-------------------
 0       | c6       | 5432 | down   | 0.500000  | standby | 0          | false             | 0
 1       | c62      | 5432 | up     | 0.500000  | primary | 0          | true              | 0
(2 rows)
#可以看出主库已挂，从库变成主库
```