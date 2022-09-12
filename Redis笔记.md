### 复制

#### 旧版复制

2.8版本之前，Redis复制实现

##### 初次同步：使用BGSAVE

从服务器 执行 slave of 主服务IP

主服务器收到 sync 请求执行 BGSAVE 操作 生成 RDB文件

并将 执行BGSAVE 之后的写操作存入缓冲区

将RDB文件发送 给从服务器，从服务器同步数据

然后将 缓冲区的写命令同步给从服务器

##### 断线重连接同步

还是使用BGSAVE 全量复制

弊端：低效能

#### 新版复制

##### 初次同步：使用BGSAVE

从服务器以前没有复制过任何服务器，或者之前执行过slaveof  no one命令

主服务器收到psync ？ -1 命令，执行BGSAVE 进行完整重同步

##### 断线重同步：

主服务器收到 psync < runid> < offset> 命令 ，返回三种结果：

+ +FULLRESYNC < runid> < offset>  回复表示执行完整重同步

+ +CONTINUE  回复表示进行部分重同步
+ -ERR 回复表示主服务器版本低于2.8 无法识别PSYNC命令，从服务将向主服务器发送SYNC

##### 部分重同步的实现涉及三部分：

+ 主从服务器的复制偏移量（replication offset）

  从服务器将自己的offset发给主服务器，主服务器通过判断offset+1的数据是否存在积压缓冲区，来决定执行何种同步操作：若在，则执行部分重同步，若不在执行完整重同步

+ 主服务器的复制积压缓冲区（replication backlog）

  固定大小（默认1MB）FIFO先进先出队列，当入队数量大于队列长度，最先入队的会被弹出

  可配置repl-backlog-size = 2 * second * write_size_per_second，

+ 主服务器的运行ID（run id）

  运行ID是服务器启动时自动生成，有40个随机十六进制字符组成。在初次复制时 主服务会将自己的运行ID传送给从服务器。再从服务器断线重新连上一个主服务器，发送之前的运行ID

  如果运行ID与连接的主服务器运行ID一致，则可继续尝试执行部分重同步，若不一致，则执行完整重同步

#### 复制的实现

1) 设置主服务器的地址和端口：SLAVEOF < master_ip>  < master_porrt>
2) 从服务器 创建套接字连接：若connect到主服务器，从服务器会为这个套接字关联一个专门用于处理复制工作的文件处理器；主服务器accept套接字后，会为该套接字创建相应的客户端状态
3) 发送PING 命令: 主服务器返回PONG
4) 身份验证： 从服务器masterauth  = 主服务器requirepass
5) 发送从服务器 监听端口号:  从服务器执行 REPLCONF listening-port < port-number>
6) 同步： 从服务器发送PSYNC命令，执行同步操作
7) 命令传播，互为客户端

#### 心跳检测

在命令传播阶段，从服务器默认会以每秒一次的频率，向主服务器发送命令：

REPLCONF ACK < replication_offset>    其中replication_offset 为从服务器当前的复制偏移量

该命令的三个作用：

+ 检测主从服务器的网络连接状态

  在主服务器发生INFO replication 命令，在从服务器列表中的lag一栏，表示从服务器最后一次发送REPLCONF ACK命令距离现在过了多少秒。一般情况lag的值在0秒或1秒，如果超过1秒说明主从之间连接出现了故障

+ 辅助实现min-slaves选项: 防止主服务器在不安全的情况下执行写命令

  min-slaves-to-write  3

  min-slaves-max-lag  10

  那么在从服务器数量少于3个或者三个从服务器的延迟（lag）都大于或等于10秒时，主服务器将拒绝执行写命令。

+ 检测命令丢失

  根据从服务器的偏移量比较主服务器偏移量，会将丢失的命令再次传播给从服务器

REPLCONF ACK命令和复制积压缓冲区都是Redis 2.8 版本新增的，在2.8版本以前，即使命令在传播过程丢失，主从服务器都不会注意

### sentinel



### cluster 集群

clusterState 

clusterNode

### 发布与订阅

subscribe < channel>、unsubscribe     

psubscribe < channel pattern>、punsubscribe

publish < channel> message

### 事务

multi  exec  watch  

### lua脚本

EVAL  "redis.call( return 'hello world ') "

script flush

script  load 

script exist

evalsha 

### 排序

sort < key> ALPHA DESC GET *-paten  LIMIT 0 4  store < key> 

### 二进制位数组

SETBIT、GETBIT、BITCOUNT、BITOP







