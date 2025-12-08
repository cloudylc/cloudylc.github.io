---
title: 为什么Redis那么快
date: 2025-10-16 19:31:29
permalink: /posts/why-is-redis-so-fast.html
index_img: /img/redis-requests-per-second.png
excerpt: Redis到底有多快？Redis快的原因？本文基于官方数据一一解答！
tags:
  - Redis
categories:
  - Redis
---

# 到底有多快
**先说结论：Redis的QPS是能够达到10万的**，当然了这也取决于机器的性能。

以上结论出自官方的文章[《How fast is Redis?》](https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/benchmarks/)，使用基准测试工具redis-benchmark验证得到。
> &nbsp;&nbsp;Redis has already been benchmarked at more than 60000 connections, and was still able to sustain 50000 q/s in these conditions. 
> &nbsp;&nbsp;Redis 已在超过 60000 个连接上进行了基准测试，在这些条件下仍然能够维持每秒 50000 次查询。

下图截选官方具体的测试结果，横轴是连接数量，纵轴是QPS：
![](redis-requests-per-second.png)

# 快的原因

机器上的性能差异不在我们的讨论范围内，单单分析是什么造就了Redis如此之快，大概可以分为以下几点。

## 存内存操作

Redis的所有数据都是存放在内存中的，这避免了磁盘的I/O瓶颈，这是速度快的**最主要的原因**。

内存与磁盘的执行速度是差别巨大的，**内存约比SSD快20倍以上，约比HDD快1万倍以上**！

下图是计算机各层级硬件执行速度：
![](rule-of-thumb-latency-numbers-letter.png)


## 单线程模型

Redis的单线程表述，可能会引起很多人的误解，特别是在当今多核CPU的发展下，都在强调程序的多线程优化。

Redis自身的任务大概分为以下几种：
- 网络I/O处理
- 键值指令读写
- 持久化
- 集群数据同步
- 其他任务
  

Redis这里所特指的单线程，并不是一个线程就把所有事情都给做了。

而是**网络I/O处理**与**键值指令读写**是由一个线程处理！(在6.x版本网络I/O读写使用多线程)

其他任务则有其他线程去处理。

而针对网络I/O处理，则使用了I/O多路复用技术解决了在单线程下的瓶颈，这部分下章节会讲述到。

既然键值指令读写是单线程处理的，为什么不使用多线程充分利用多核CPU的特性？

官方在一篇[《Redis FAQ》](https://redis.io/docs/latest/develop/get-started/faq/)的文章中回答了该问题：
> &nbsp;&nbsp;It's not very frequent that CPU becomes your bottleneck with Redis, as usually Redis is either memory or network bound. For instance, when using pipelining a Redis instance running on an average Linux system can deliver 1 million requests per second, so if your application mainly uses O(N) or O(log(N)) commands, it is hardly going to use too much CPU.
> &nbsp;&nbsp;CPU 很少成为 Redis 的瓶颈，因为 Redis 通常受内存或网络的限制。例如，当使用管道时，在普通 Linux 系统上运行的 Redis 实例每秒可以处理 100 万个请求，因此如果您的应用程序主要使用 O(N) 或 O(log(N)) 命令，它几乎不会占用太多 CPU。

说白了，单线程就在性能上的表现已经非常足够了，而且在6.x版本，网络I/O读写这一部分也交给多线程去完成。
键值指令读写就真的是单线程去完成，彻底的解放了，而且在代码的实现上，它更加容易使得逻辑更清晰。

不强行使用多线程，还有很重要的一点，那就是多线程它是有成本的！
1. 创建新线程的开销
2. 线程上下文切换的开销
3. 线程间竞争锁开销

所以当一个任务足够复杂足够耗时时，使用多线程的收益是远远大于成本的。
当一个任务足够简单不耗时，强行使用多线程反而会出现成本大于收益，严重的得不偿失！

键值指令读写是单线程处理还有一个好处，就是它能保证命令间的**原子性**。
因为命令是按序串行执行，不会随机跳转执行。

## I/O多路复用机制
Redis 使用I/O多路复用技术处理并发连接，采用 epoll + 自行实现的简单事件框架。
当socket准备好执行`accept`/`read`/`write`/`close`时，都会产生一个事件,然后利用 epoll 的多路复用特性，绝不在IO上浪费时间。
![](redis-io.png)

## 高效的数据结构
Redis有5种基本类型的数据结构：`String`、`List`、`Hash`、`Set`、`Sorted Set`
但每种数据结构都内自己的多种底层内部编码实现，如下图所示：
![](redis-type-encoding.png)

这样做的好处
1. 在对外数据结构和命令没有改变的情况下，可以改进内部编码提升性能，用户无感
2. 内部多种实现可以在不同场合下发挥不同的优势

