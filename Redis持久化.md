> Redis 持久化
- Redis 支持RDB和AOF两种持久化机制，持久化功能有效地避免因进程退出造成的数据丢失问题，当下次重启时利用之前持久化的文件即可实现数据恢复

> RDB

- RDB 持久化是把当前进程数据生成快照保存到硬盘的过程，触发RDB持久化过程分为手动触发和自动触发
- 手动触发分别对应save和bgsave命令:
- - 1. save命令: 阻塞当前Redis服务器，指定RDB过程完成为止，对于内存比较大的实例会造成长时间阻塞，线上环境不建议使用.
- - redis-cli
  
![avator](images/save_cli.png)

- - redis-server

![avator](images/save_server.png)

- - 2. bgsave命令：Redis进程执行fork操作创建子进程，RDB持久化过程有子进程负责，完成后自动结束。阻塞只发生在fork阶段，一般时间很短

- - redis-cli

![avator](images/bgsave_cli.png)

- - redis-server

![avator](images/bgsave_server.png)

- Redis内部所有涉及RDB的操作都采用bgsave的方式
- 自动触发RDB的持久化机制：
- - 1. 使用save相关配置，如“save m n ",表示m秒内数据集存在n次修改时，自动触发bgsave
- - 2. 如果从节点执行全量复制操作，主节点自动执行bgsave生成RDB文件发送个从节点
- - 3. 执行debug reload 命令重新加载Redis时，也会自动触发save操作
- - 4. 默认情况下执行shutdown命令时，如果没有开启AOF持久化功能则自动执行bgsave

> bgsave 持久化流程

![avator](images/bgsave_step.png)

> RDB 文件的处理

- 保存：RDB 文件保存在dir配置指定的目录下，文件名通过dbfilename配置指定。可以通过执行config set dir{new Dir} 和 config set dbfilename{new FileName} 运行期动态执行，当下次运行时RDB文件会保存到新目录。

![avator](images/redis_config_dbfile.png)

- 当遇到坏盘或磁盘写满等情况时，可以通过 config set dir{new Dir}在线修改文件路径到可用的磁盘路径，之后执行bgsave进行磁盘切换，同一适用于AOF持久化文件   
- 压缩：Redis默认采用LZF算法对生成的RDB文件做压缩处理，压缩后的文件远远小于内存大小，默认开启，可用通过参数 config set rdbcompression{yes|no} 动态修改，配置文件为：

![avator](images/redis_com.png)

- 校验：如果Redis 加载损坏的RDB文件时拒绝启动，可用使用Redis提供的redis-check-dump工具检测RDB文件并获取对应的错误报告

> AOF 

- AOF(append only file) 持久化：以独立日志的方式记录每次写命令，重启时在重新执行AOF文件中的命令达到恢复数据的目的
- 作用：解决了数据持久化的实时性