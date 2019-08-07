> 阻塞

- 参考网址 ： http://ghoulich.xninja.org/2016/10/15/how-to-monitor-redis-status/

- 单线程的Redis处理命令时只能使用一核CPU。而CPU饱和是指Redis把单核CPU使用率跑到接近100%

- redis-cli -- bigkeys
- 根据结果汇总信息获取大对象的键，以及不同类型数据结构的使用情况.


```
root@FM:/home/software/6380/redis-5.0.5.6380# redis-cli --bigkeys
# 扫描整个键值空间，查找最大的键值以及每种键值类型的平均大小
# 你可以使用-i 0.1  来休眠扫描命令(每0.1秒100条)
# Scanning the entire keyspace to find biggest keys as well as
# average sizes per key type.  You can use -i 0.1 to sleep 0.1 sec
# per 100 SCAN commands (not usually needed).

[00.00%] Biggest list   found so far 'mylist' with 4 items
[00.00%] Biggest string found so far 'java' with 5 bytes
[00.00%] Biggest hash   found so far 'carnival' with 3 fields
[00.00%] Biggest string found so far 'python' with 8 bytes
[00.00%] Biggest string found so far 'key' with 128 bytes

-------- summary -------

Sampled 7 keys in the keyspace!
Total key length in bytes is 39 (avg len 5.57)

Biggest string found 'key' has 128 bytes
Biggest   list found 'mylist' has 4 items
Biggest   hash found 'carnival' has 3 fields

5 strings with 147 bytes (71.43% of keys, avg size 29.40)
1 lists with 4 items (14.29% of keys, avg size 4.00)
0 sets with 0 members (00.00% of keys, avg size 0.00)
1 hashs with 3 fields (14.29% of keys, avg size 3.00)
0 zsets with 0 members (00.00% of keys, avg size 0.00)
0 streams with 0 entries (00.00% of keys, avg size 0.00)
```

- redis-cli -h{ip} -p{port} -- stat
- 获取当前Redis的使用情况，该命令每秒输出一行统计信息

```
root@FM:/home/software/6380/redis-5.0.5.6380# redis-cli --stat
------- data ------ --------------------- load -------------------- - child -
keys       mem      clients blocked requests            connections          
7          808.77K  1       0       72 (+0)             12          
7          808.77K  1       0       73 (+1)             12          
7          808.77K  1       0       74 (+1)             12          
7          808.77K  1       0       75 (+1)             12          
7          808.77K  1       0       76 (+1)             12          
7          808.77K  1       0       77 (+1)             12          
7          808.77K  1       0       78 (+1)             12          
7          808.77K  1       0       79 (+1)             12          
```

- info commandstats 
- 分析出命令不合理开销时间

```
耗时统计：

127.0.0.1:6379> info commandstats
# Commandstats
cmdstat_incr:calls=1,usec=7,usec_per_call=7.00
cmdstat_mset:calls=1,usec=8,usec_per_call=8.00
cmdstat_set:calls=2,usec=1749,usec_per_call=874.50
cmdstat_get:calls=2,usec=28,usec_per_call=14.00
cmdstat_strlen:calls=10,usec=6,usec_per_call=0.60
cmdstat_save:calls=1,usec=2214,usec_per_call=2214.00
cmdstat_client:calls=6,usec=112,usec_per_call=18.67
cmdstat_command:calls=7,usec=3337,usec_per_call=476.71
cmdstat_type:calls=14,usec=11,usec_per_call=0.79
cmdstat_scan:calls=2,usec=3783,usec_per_call=1891.50
cmdstat_mget:calls=1,usec=6,usec_per_call=6.00
cmdstat_keys:calls=1,usec=14,usec_per_call=14.00
cmdstat_llen:calls=2,usec=4,usec_per_call=2.00
cmdstat_info:calls=25,usec=2688,usec_per_call=107.52
cmdstat_dbsize:calls=3,usec=1312,usec_per_call=437.33
cmdstat_bgsave:calls=1,usec=413,usec_per_call=413.00
cmdstat_hlen:calls=2,usec=530,usec_per_call=265.00
```

> 持久化阻塞

- 持久化引起主线程阻塞的操作主要有:fork阻塞、AOF刷盘阻塞、HugePage写操作阻塞.
- fork 阻塞
- fork 操作发生在RDB和AOF重写时，Redis主线程调用fork操作产生共享内存的子进程，由子进程完成持久化文件重写工作。如果fork操作本身耗时过长，必然会导致主线程的阻塞。
- info stats 命令获取到latest_fork_usec指标，表示Redis最近一次fork操作耗时

```
127.0.0.1:6379> info stats
# Stats
total_connections_received:13                     //Redis 服务器接收的连接总数
total_commands_processed:86                       //Redis 服务器处理的命令总数
instantaneous_ops_per_sec:0                       //每秒钟处理的命令数量
total_net_input_bytes:1952                        //通过网络接收的数据总量，以字节为单位
total_net_output_bytes:137723                     //通过网络发送的数据总量，以字节为单位
instantaneous_input_kbps:0.00                     //每秒钟接收数据的速率，以kbps为单位
instantaneous_output_kbps:0.00                    //每秒钟发送数据的速率，以kbps为单位
rejected_connections:0                            //Redis服务器由于maxclients限制而拒绝的连接数量
sync_full:0                                       //Redis主从完全同步的次数
sync_partial_ok:0                                 //Redis 服务器接收PSYNC请求的次数
sync_partial_err:0                                //Redis 服务器拒绝PSYNC请求的次数
expired_keys:0                                    //键过期事件的总数
evicted_keys:0                                    //由于maxmemory限制，而被回收内存的键的总数
keyspace_hits:31                                  //在字典中成功查找到的键的次数
keyspace_misses:1                                 //在字典中未能成功查找到键的次数
pubsub_channels:0                                 //客户端订阅的发布/订阅频道的总数量
pubsub_patterns:0                                 //客户端订阅的发布/订阅模式的总数量
latest_fork_usec:2030                             //最近一次fork操作消耗的时间，以微妙为单位
migrate_cached_sockets:0                          //迁移已缓存的套接字的数量
slave_expires_tracked_keys:0
active_defrag_hits:0
active_defrag_misses:0
active_defrag_key_hits:0
active_defrag_key_misses:0
```
- AOF 刷盘阻塞
- 当我们开启AOF持久化功能时，文件刷盘的方式一般采用每秒一次，后台线程每秒对AOF文件做fsync操作。当硬盘压力过大时，fsync操作需要等待，直到写入完成。如果主线程发现距离上一次的fsync成功超过2秒，为了数据数据安全性它会阻塞直到后台线程执行fsync操作完成。这种阻塞行为主要是硬盘压力引起的.

```
127.0.0.1:6379> info persistence
# Persistence
loading:0                               //表示Redis是否正在加载一个转储文件的标志
rdb_changes_since_last_save:0           //从最近一次转储至今，RDB的修改次数
rdb_bgsave_in_progress:0                //表示Redis正在保存RDB的标志
rdb_last_save_time:1565179669           //最近一次成功保存RDB的时间戳，基于Epoch时间
rdb_last_bgsave_status:ok               //最近一次RDB保存操作的状态
rdb_last_bgsave_time_sec:0              //最近一次RDB保存操作消耗的时间，以秒为单位。
rdb_current_bgsave_time_sec:-1          //如果Redis正在执行RDB保存操作，那么这个字段表示已经消耗的时间，以秒为单位。
rdb_last_cow_size:311296
aof_enabled:0                           //表示Redis是否启用AOF日志功能的标志。
aof_rewrite_in_progress:0               //表示Redis是否正在执行一次AOF重写操作的标志。
aof_rewrite_scheduled:0                 //表示一旦Redis正在执行的RDB保存操作完成之后，是否就会调度执行AOF重写操作的标志
aof_last_rewrite_time_sec:-1            //最近一次AOF重写操作消耗的时间，以秒为单位
aof_current_rewrite_time_sec:-1         //如果Redis正在执行AOF重写操作，那么这个字段表示已经消耗的时间，以秒为单位
aof_last_bgrewrite_status:ok            //最近一次AOF重写操作的状态
aof_last_write_status:ok
aof_last_cow_size:0
```

- HugePage 写操作阻塞
- 子进程在执行重写期间利用Linux写时复制技术降低内存消耗，因此只有写操作时Redis才复制要修改的内存页。对于开启Transparent HugePage的操作系统，每次写命令引起的复制内存单位由4K变为2MB，放大了512倍，会拖慢写操作的执行时间，导致大量写操作慢查询。

> CPU 竞争
- 进程竞争:Redis 是典型的CPU密集型应用，不建议和其他多核CPU密集型服务部署在一起。当其他进程过度消耗CPU时，将严重影响Redis吞吐量。
- 绑定CPU:部署Redis时为了充分利用多核CPU，通常一台机器部署多个实例。常见的一种优化是把Redis进程绑定到CPU上，用于降低CPU频繁上下文切换的开销
- 当Redis父进程创建子进程进行RDB/AOF重写时，如果做了CPU绑定，会与父进程共享使用一个CPU。子进程重写时对单核CPU使用率通常在90%以上，父进程与子进程将产生激烈CPU竞争，极大影响Redis稳定性.因此对于开启了持久化或参与复制的主节点不建议绑定CPU