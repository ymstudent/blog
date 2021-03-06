---
title: "Redis AOF持久化"
date: 2021-01-03T21:25:17+08:00
draft: false
tags: ["redis"]
keywords: ["redis", "AOF持久化"]
---

Redis AOF 是以文本日志的形式记录 Redis 处理的每一个写入或删除操作。

AOF 对日志文件的**写入操作使用的追加模式，有灵活的同步策略，支持每秒同步、每次修改同步和不同步，缺点就是相同规模的数据集，AOF 要大于 RDB，AOF 在运行效率上往往会慢于 RDB。**

### AOF持久化的实文件

AOF 持久化可以分为三个步骤：命令追加（append）， 文件写入（write）， 文件同步（sync）

#### 命令追加

当 AOF 持久化打开时，服务器在执行完一个写命令后，会以**协议格式**将被执行的写命令追加到服务器的 aof_buf 缓冲区的末尾。

eg：SET KEY VALUE 命令会被以：*3\r\n$3\r\nSET\r\nKEY\r\nVALUE\r\n 这样的的格式加入到 aof_buf 缓冲区的末尾。

#### 文件写入

redis 的服务器进程就是一个事件循环（loop）, 这个事件循环的中的文件事件负责接收客户端的请求命令。以及向客户端发送命令回复，而时间事件则负责执行向 serverCron 函数这样需要定时运行的函数。

服务器每次结束一个事件循环之前都会调用 flushAppendOnlyFile 函数，根据 appendfsync 配置考虑是否需要将 aof_buf 缓冲区的内容写入和保存到 AOF 文件中。

| appendfsync 选项的值 | flushAppendOnlyFlile 函数的行为                              |
| -------------------- | ------------------------------------------------------------ |
| always               | 将 aof_buf 缓冲区所有的内容写入并同步到 AOF 文件             |
| everysec（默认选项） | 将 aof_buf 缓冲区的所有内容写入到 AOF 文件，如果上次同步 AOF 的时间距离现在超过 1s，则再次对 AOF 文件进行同步，同步操作由一个专门的线程负责执行 |
| no                   | 将 aof_buf 缓冲区的内容写入到 AOF 文件，但不对 AOF 文件进行同步，何时同步由操作系统来决定 |

#### 文件同步

在现代操作系统中，为了提高文件的写入效率，当用户调用 write 函数，将一些数据写入到文件的时候，操作系统通常会将写入数据暂时保存在一个内存缓冲区中，等到缓冲区的空间被填满，或者是超过指定的时限后，才真正的将缓冲区中的数据写入磁盘里面。

为了防止写入缓冲区丢失，操作系统提供了 fsync 和 fdatasync 两个同步函数，它们可以强制让缓冲区中的数据写入到硬盘中，从而确保写入数据的安全性。

### AOF文件的载入与数据还原

因为 AOF 文件包括了重建数据库状态所需的所有写命令，所以服务器只要读入并重新执行一遍 AOF 文件保存的写命令，就能还原服务器关闭之前的数据库状态。

* 创建一个不带网络连接的伪客户端。因为 redis 命令只能在客户端上下文中执行，而载入 AOF 文件时所使用的命令直接来源于 AOF 文件而不是网络连接，所以服务器使用了一个没有网络连接的伪客户端来执行 AOF 文件保存的命令

* 从 AOF 文件中分析并读取出一条写命令

* 使用伪客户端执行被读出的写命令

* 一直执行2、3步骤，直到 AOF 文件中的所有写命令都被处理完毕为止

### AOF重写

为了解决 AOF 文件体积膨胀问题，redis 提供了 AOF 文件重写 ( rewrite ) 功能。

#### 重写的实现

AOF 文件重写不需要对现有的 AOF 文件进行任何读取、分析或者写入操作，而是通过读取服务器当前数据库的状态来实现的：从数据库中读取键现在的值，然后用一条命令去记录键值对，代替之前记录这个键值对的多条命令。

**需要注意的是**： 在实际中，为了避免在执行命令时造成客户端的输入缓冲区溢出，重写程序在处理列表，哈希表，集合，有序集合这四种可能会包含多个元素的键时，会先检查键所包含的元素数量是否超过了 redis.h/REDIS_AOF_REWRITE_ITEMS_PER_CMD 常量的值（默认为64），那么重写程序会使用多条命令来记录键的值。

#### 后台重写（BGREWRITEAOF）

AOF 重写操作会有大量的写入操作，执行重写的线程将会被长时间的阻塞，所以 redis 将 AOF 重写放到子进程中执行。

后台重写的优势：

* 子进程进行 AOF 重写期间，服务器主进程可以继续处理命令请求。

* 子进程带有服务器主进程的数据副本，使用子进程而不是线程，可以在避免使用锁的情况下，保证数据的安全性。

后台重写导致的问题：

* 重写期间，服务器可能接受到新的写命令，从而使得服务器当前的数据库状态和重写后的AOF文件所保存的数据库状态不一致。

解决数据不一致问题的方法：

* 设置 AOF 重写缓冲区。从服务器创建重写子进程开始，所有的写命令不仅会被追加到 AOF 缓冲区，还会被追加到 AOF 重写缓冲区。在子进程完成重写后，父进程会将 AOF 重写缓冲区的所有内容追加到新 AOF 文件的末尾，使新 AOF 文件的内容与当前数据库状态保持一致。最后服务器使用新的 AOF 文件替换掉旧的 AOF 文件，完成 AOF 重写操作。