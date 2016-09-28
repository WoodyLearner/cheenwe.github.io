---
layout: post
title:  Redis 持久化与备份策略
tags: redis backup
category: redis
---


# 持久化

redis 是内存数据库，掉电会丢失，转移数据不便。持久化就是内存数据到硬盘数据的转化。 当然，也可以硬盘到内存（备份的概念，保存，恢复）。


## 实现

两种方法: 快照方式（rdb）+日志方式（aof） 快速+最大化redis性能+方便: rdb 模式 更持久: aof 模式

### rdb 快照模式

- Snapshotting (快照) 语法

快照是默认的持久化方式（内存全拷贝）。这种方式是就是将内存中数据以快照的方式写入到二进制文件中,默认的文件名为dump.rdb。可以通过配置设置自动做快照持久 化的方式。我们可以配置redis在n秒内如果超过m个key被修改就自动做快照，下面是默认的快照保存配置。（建议从下往上看，60s-300s-900s）

```sh
    save 900 1  //900秒内如果超过1个key被修改，则发起快照保存
    save 300 10 //300秒内容如超过10个key被修改，则发起快照保存
    save 60 10000  //(这3个选项都屏蔽,则rdb禁用)

    stop-writes-on-bgsave-error yes  // 后台备份进程出错时,主进程停不停止写入
    rdbcompression yes    // 导出的rdb文件是否压缩
    Rdbchecksum   yes //  导入rbd恢复时数据时,要不要检验rdb的完整性
    dbfilename dump.rdb  //导出来的rdb文件名
    dir /var/lib/redis/  //rdb的放置路径
```

- 详细的快照保存过程

1.redis调用fork,现在有了子进程和父进程。

2.父进程继续处理client请求，子进程负责将内存内容写入到临时文件。由于os的写时复制机制（copy on write)父子进程会共享相同的物理页面，当父进程处理写请求时os会为父进程要修改的页面创建副本，而不是写共享的页面。所以子进程的地址空间内的数 据是fork时刻整个数据库的一个快照。

3.当子进程将快照写入临时文件完毕后，用临时文件替换原来的快照文件，然后子进程退出。

client 也可以使用save或者bgsave命令通知redis做一次快照持久化。save操作是在主线程中保存快照的，由于redis是用一个主线程来处理所有 client的请求，这种方式会阻塞所有client请求。所以不推荐使用。另一点需要注意的是，每次快照持久化都是将内存数据完整写入到磁盘一次，并不是增量的只同步增加数据。如果数据量大的话，而且写操作比较多，必然会引起大量的磁盘io操作，可能会严重影响性能。

另外由于快照方式是在一定间隔时间做一次的，所以如果redis意外down掉的话，就会丢失最后一次快照后的所有修改。如果应用要求不能丢失任何修改的话，可以采用aof持久化方式。

### aof 日志模式

aof 比快照方式有更好的持久化性，是由于在使用aof持久化方式时,redis会将每一个收到的写命令都通过write函数追加到文件中(默认是 appendonly.aof)。当redis重启时会通过重新执行文件中保存的写命令来在内存中重建整个数据库的内容。当然由于os会在内核中缓存 write做的修改，所以可能不是立即写到磁盘上。这样aof方式的持久化也还是有可能会丢失部分修改。不过我们可以通过配置文件告诉redis我们想要 通过fsync函数强制os写入到磁盘的时机。有三种方式如下（默认是: 每秒fsync一次）

```sh
appendonly yes              //启用aof持久化方式
# appendfsync always      //每收到写命令就立即强制写入磁盘，最慢的，但是保证完全的持久化，不推荐使用
appendfsync everysec     //每秒钟强制写入磁盘一次，在性能和持久化方面做了很好的折中，推荐
# appendfsync no         //完全依赖os，性能最好,持久化没保证（操作系统自身的同步）
no-appendfsync-on-rewrite  yes  //正在导出rdb快照的过程中,要不要停止同步aof
auto-aof-rewrite-percentage 100  //aof文件大小比起上次重写时的大小,增长率100%时,重写
auto-aof-rewrite-min-size 64mb //aof文件,至少超过64M时,重写
```

重写：内存中有N多数据，是NNN次语句得到的，为了节省日志文件空间，将内存数据转化为直接优化过的语句，不是以前的（详细）操作步骤，但是达到的“数据”效果是一样的。 aof 的方式也同时带来了另一个问题。持久化文件会变的越来越大。例如我们调用incr test命令100次，文件中必须保存全部的100条命令，其实有99条都是多余的。因为要恢复数据库的状态其实文件中保存一条set test 100就够了。为了压缩aof的持久化文件。redis提供了 bgrewriteaof 命令。收到此命令redis将使用与快照类似的方式将内存中的数据 以命令的方式保存到临时文件中，最后替换原来的文件。aof 过程：

```sh
1.redis调用fork ，现在有父子两个进程

2.子进程根据内存中的数据库快照，往临时文件中写入重建数据库状态的命令

3.父进程继续处理client请求，除了把写命令写入到原来的aof文件中。同时把收到的写命令缓存起来。这样就能保证如果子进程重写失败的话并不会出问题。

4.当子进程把快照内容写入已命令方式写到临时文件中后，子进程发信号通知父进程。然后父进程把缓存的写命令也写入到临时文件。

5.现在父进程可以使用临时文件替换老的aof文件，并重命名，后面收到的写命令也开始往新的aof文件中追加。

需要注意到是重写aof文件的操作，并没有读取旧的aof文件，而是将整个内存中的数据库内容用命令的方式重写了一个新的aof文件,这点和快照有点类似。
```

注意：

* aof和rdb同时存在的时候，有限使用aof恢复数据；

* aof和rdb恢复的时候，谁快，rdb快，直接拷贝数据，aof还要执行语句；

* 建议同时使用aof 和 rdb

* aof 文件 和 rdb文件 在下次redis启动的时候会执行或载入，达到备份的目的。


## SAVE
Redis SAVE命令用来创建备份当前Redis数据库。

### 备份 SAVE

```sh
127.0.0.1:6379> SAVE

OK
```