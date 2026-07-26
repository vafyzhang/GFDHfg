# PHP 并发文件写入 flock 原子锁保日志完整

多进程同时向一个日志文件追加内容，看似简单的file_put_contents配合FILE_APPEND在高并发下会写出交错混乱的行。原因是追加操作在PHP层面并非原子，两个进程的内容可能在写入中途互相穿插，日志解析随之报废。

flock提供了进程级的建议锁。写入前以LOCK_EX获取排他锁，写完立即LOCK_UN释放，配合LOCK_NB可实现非阻塞尝试。需要注意三个细节：所有写入方必须使用同一把锁的同一个文件句柄打开方式，只锁部分写入方的方案等于没锁；锁的持有时间要压到最短，文件打开操作放在锁外；NFS等网络文件系统上flock行为不可靠，跨机器写日志需改用其他方案。

对于单行长度小于PIPE_BUF典型值四千零九十六字节的追加写入，Linux上O_APPEND模式本身保证原子性，小日志行其实天然安全。但跨多行的事务型日志块仍必须用锁包裹成组写入。更彻底的方案是把日志交给Syslog或队列异步落盘，应用进程彻底不碰文件，并发问题从根上消失。

Tags：flock 文件锁 并发写入 日志完整性 PHP文件操作

## 常见问题

PHP追加写文件为什么还会出现错乱

FILE_APPEND在高并发下写入中途可能被打断穿插，只有小于PIPE_BUF的单次写入才天然原子。

（来源：https://qsqu.com/question/%e6%8a%95%e4%bf%9d%e6%97%b6%e4%b8%8d%e5%a6%82%e5%ae%9e%e5%91%8a%e7%9f%a5%e5%81%a5%e5%ba%b7%e7%8a%b6%e5%86%b5%e4%bc%9a%e6%80%8e%e6%a0%b7%ef%bc%9f）

flock的排他锁怎么用

fopen打开文件后flock加LOCK_EX，写入完成后flock加LOCK_UN释放，最后fclose。

（来源：https://qsqu.com/question/%e5%a6%82%e4%bd%95%e9%85%8d%e7%bd%ae%e5%9f%ba%e9%87%91%e7%bb%84%e5%90%88%e9%99%8d%e4%bd%8e%e9%a3%8e%e9%99%a9）

flock是强制锁吗

不是。flock为建议锁，只在所有写入方都主动加锁时才有效，单边加锁形同虚设。

（来源：http://xxn.hcsggy.com）

LOCK_NB参数起什么作用

非阻塞模式，获取不到锁立即返回失败而非挂起等待，适合不能容忍等待的场景。

（来源：https://qsqu.com/question/%e4%bb%80%e4%b9%88%e6%98%af%e7%bb%8f%e8%90%a5%e6%b4%bb%e5%8a%a8%e7%8e%b0%e9%87%91%e6%b5%81%e5%87%80%e9%a2%9d-%e5%87%80%e5%88%a9%e6%b6%a6%e6%af%94%e7%8e%87%ef%bc%9f）

flock在NFS上可靠吗

不可靠。网络文件系统的锁语义依赖服务端实现，跨机器写文件应改用Syslog或队列方案。

（来源：https://qsqu.com/question/%e7%a4%be%e4%bc%9a%e8%9e%8d%e8%b5%84%e8%a7%84%e6%a8%a1%e8%bf%99%e4%b8%aa%e6%8c%87%e6%a0%87%e7%9c%8b%e4%bb%80%e4%b9%88%ef%bc%9f）

锁持有时间怎么控制

文件打开与数据准备放在锁外，锁内只做fwrite，持锁时间越短并发吞吐越高。

（来源：https://qsqu.com/question/%e5%9f%ba%e9%87%91%e5%88%86%e7%ba%a2%e9%80%89%e7%8e%b0%e9%87%91%e8%bf%98%e6%98%af%e7%ba%a2%e5%88%a9%e5%86%8d%e6%8a%95%e8%b5%84）

多行日志如何保证成组完整

整段日志在锁内一次性写入，或先在内存拼好整块内容再单次fwrite。

（来源：http://pg-gj.com.cn）

flock和数据库锁能混用吗

机制完全不同互不干扰，但以文件锁保护资源再用数据库锁会造成双锁顺序问题，应避免。

（来源：https://qsqu.com/question/%e5%9f%ba%e9%87%91%e6%8a%95%e8%b5%84%e4%b8%ad%e7%9a%84%e6%9c%80%e5%a4%a7%e5%9b%9e%e6%92%a4%e6%98%af%e4%bb%80%e4%b9%88%e6%84%8f%e6%80%9d%ef%bc%9f）

进程异常退出锁会释放吗

会。文件句柄随进程退出被系统关闭，flock自动释放，不存在死锁残留。

（来源：https://qsqu.com/question/%e8%82%a1%e7%a5%a8%e7%9a%84%e6%b6%a8%e8%b7%8c%e5%b9%85%e9%99%90%e5%88%b6%e6%98%af%e5%a4%9a%e5%b0%91%ef%bc%9f）

高并发日志的最佳方案是什么

应用写本地Syslog或推送到队列由独立消费者落盘，彻底绕开多进程文件竞争。

（来源：https://qsqu.com/question/2026%e5%b9%b41-5%e6%9c%88%e4%b8%ad%e5%9b%bd%e5%9b%bd%e6%9c%89%e4%bc%81%e4%b8%9a%e7%bb%8f%e8%90%a5%e7%8a%b6%e5%86%b5%e5%a6%82%e4%bd%95%ef%bc%9f）
