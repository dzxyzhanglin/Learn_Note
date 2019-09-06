# Redis学习笔记



## Windows版本



### Redis启动

`redis-server.exe  <配置文件>`

> 不加配置文件也可以，redis会以默认的配置文件启动

### Redis配置

* `requirepass <>` 设置redis访问密码，为空表示不设置密码
* `maxmemory <byte>` 设置redis最大内存。如`maxmemory 5GB`





## Redis常用命令

* `auth <密码>` ： redis通过命令行访问设置了密码的redis。

* `info memory` ： 查看redis内存使用情况

  > `used_memory` ： 由 Redis 分配器分配的内存总量，包含了redis进程内部的开销和数据占用的内存，以字节（byte）为单位
  >
  > `used_memory_human` ： 以更直观的单位展示分配的内存总量。
  >
  > `used_memory_rss` ： 向操作系统申请的内存大小。与 top 、 ps等命令的输出一致。
  >
  > `used_memory_rss_human` ： 以更直观的单位展示向操作系统申请的内存大小。
  >
  > `used_memory_peak` ： redis的内存消耗峰值(以字节为单位)
  >
  > `used_memory_peak_human` ： 以更直观的格式返回redis的内存消耗峰值
  >
  > `used_memory_peak_perc` ： 使用内存达到峰值内存的百分比，即(used_memory/ used_memory_peak) *100%
  >
  > `used_memory_overhead` ： Redis为了维护数据集的内部机制所需的内存开销，包括所有客户端输出缓冲区、查询缓冲区、AOF重写缓冲区和主从复制的backlog。
  >
  > `used_memory_startup` ： Redis服务器启动时消耗的内存
  >
  > `used_memory_dataset` ： 数据占用的内存大小，即used_memory-sed_memory_overhead
  >
  > `used_memory_dataset_perc` ： 数据占用的内存大小的百分比，100%*(used_memory_dataset/(used_memory-used_memory_startup))
  >
  > `total_system_memory` ： 整个系统内存
  >
  > `total_system_memory_human` ： 以更直观的格式显示整个系统内存
  >
  > `used_memory_lua` ： Lua脚本存储占用的内存
  >
  > `used_memory_lua_human` ： 以更直观的格式显示Lua脚本存储占用的内存
  >
  > `maxmemory` ： Redis实例的最大内存配置
  >
  > `maxmemory_human` ： 以更直观的格式显示Redis实例的最大内存配置
  >
  > `maxmemory_policy` ： 当达到maxmemory时的淘汰策略
  >
  > `mem_fragmentation_ratio` ： 碎片率，used_memory_rss/ used_memory
  >
  > `mem_allocator` ： 内存分配器
  >
  > `active_defrag_running` ： 表示没有活动的defrag任务正在运行，1表示有活动的defrag任务正在运行（defrag:表示内存碎片整理）
  >
  > `lazyfree_pending_objects` ： 0表示不存在延迟释放的挂起对象

* `select <index>` ： 切换数据库