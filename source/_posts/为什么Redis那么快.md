---
title: 为什么Redis那么快
date: 2025-10-16 19:31:29
index_img: /img/redis-requests-per-second.png
tags:
  - Redis
categories:
  - Redis
---

# 到底有多快
根据官方的文章[《How fast is Redis?》](https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/benchmarks/)，使用基准测试工具redis-benchmark验证，**Redis的QPS是能够达到10万的**。

下图截选具体的测试结果，横轴是连接数量，纵轴是QPS。
>Redis has already been benchmarked at more than 60000 connections, and was still able to sustain 50000 q/s in these conditions. ——[《How fast is Redis?》](https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/benchmarks/)

![](redis-requests-per-second.png)

# 快的原因

| 核心因素    | 具体实现                                      |
| ------- | ----------------------------------------- |
| 纯内存操作   | 数据存储在内存中，避免了磁盘I/O瓶颈，这是速度快的 **根本**         |
| 单线程模型   | 核心命令处理是单线程，避免了多线程的 **上下文切换** 和 **锁竞争** 开销 |
| I/O多路复用 | 使用  epoll 等技术，单线程高效监听和处理数万个网络连接           |
| 高效的数据结构 | 为不同的数据类型专门优化了底层实现                         |
| 优化的网络协议 | 采用简单的 RESP (Redis序列化协议) ，解析效率高，减少网络开销     |

