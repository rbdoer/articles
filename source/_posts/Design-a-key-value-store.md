---
title: Design a key value store
date: 2017-09-11 14:09:17
tags:
categories: 系统设计
---

## 设计缓存系统

### 需要考虑的问题

	- LRU
	- 回收策略
	- 并发问题
	- 分布式系统

### LRU

	hashtable + 双向链表实现
	- If A exists in the cache, we just return immediately.
	- If not and the cache has extra storage slots, we fetch resource A and return to the client. In addition, insert A into the cache.
	- If the cache is full, we kick out the resource that is least recently used and replace it with resource A.

### 回收策略

	- Random Replacement (RR) – As the term suggests, we can just randomly delete an entry.
	- Least frequently used (LFU) – We keep the count of how frequent each item is requested and delete the one least frequently used.
	- W-TinyLFU – I’d also like to talk about this modern eviction policy. In a nutshell, the problem of LFU is that sometimes an item is only used frequently in the past, but LFU will still keep this item for a long while. W-TinyLFU solves this problem by calculating frequency within a time window. It also has various optimizations of storage.

### 并发访问

	解决 reader-writer problem （加锁|cas）

### 分布式

	选择hash算法（一致性hash等）

### 备份

	append log模式进行主从同步等


## 设计K-V存储系统

### Basic key-value storage

	基本的hash表实现，如果存储的数据太大，两种解决方案：
	- Compress your data. This should be the first thing to think about and often there are a bunch of stuff you can compress. For example, you can store reference instead of the actual data. You can also use float32 rather than float64. In addition, using different data representations like bit array (integer) or vectors can be effective as well.
	- Storing in a disk. When it’s impossible to fit everything in memory, you may store part of the data into disk. To further optimize this, you can think of the system as a cache system. Frequently visited data is kept in memory and the rest is on a disk.

### Distributed key-value storage & Sharding

	分布式设计，同分布是缓存系统设计


### System availability

	The most common solution is replica. By setting machines with duplicate resources, we can significantly reduce the system downtime. If a single machine has 10% of chance to crash every month, then with a single backup machine, we reduce the probability to 1% when both are down

### Consistency

	repica + consistency 会出现一个主从数据不一致的问题，怎么解决？
	业界通常的做法是所有操作写入commit log，通过commit log来做数据同步，commit log还可以做断电数据恢复等操作

### Read throughput

	如何提升系统吞吐量


参考资料

* [design-a-cache-system](http://blog.gainlo.co/index.php/2016/05/17/design-a-cache-system/)
* [design-a-key-value-store-part-i](http://blog.gainlo.co/index.php/2016/06/14/design-a-key-value-store-part-i)
* [design-key-value-store-part-ii](http://blog.gainlo.co/index.php/2016/06/21/design-key-value-store-part-ii)