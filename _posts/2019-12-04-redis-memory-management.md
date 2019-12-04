---
title: Redis内存管理
key: 10020
tags: Redis
layout: article
category: blog
comment: true
date: 2019-12-02 23:48:00 +08:00
modify_date: 2019-12-02 23:48:00 +08:00
---

本文源于LeetCode上遇到的一道题[LRU Cache](https://leetcode.com/problems/lru-cache/), 第一次听说LRU这个词还是在大三操作系统课上, 操作系统在发生缺页异常之后会进行页面置换, LRU正是几个页面置换算法之一。正巧, Redis的内存管理也用到了LRU算法, 故借此机会拓展开来阅读一下Redis内存管理的源码以及详细介绍一下LRU算法的实现。

本文参考的源码是Redis 3.0

## 配置项说明

`redis.conf`中有两个和内存管理相关的配置项: `maxmemory` 和 `maxmemory-policy`。这两个配置的主要意义在于控制内存的精细化使用，尤其适用于冷热数据较为明显的业务场景。如读写比超过10:1的高并发缓存系统等

### maxmemory

`maxmemory` 定义了可供使用的最大内存字节数上限, 当达到设定的阈值后, Redis会根据设定的`maxmemory-policy`内存驱逐策略对keys进行清理.

有两个比较值得关注的点:
- 缺省值是0
- 针对32位机器内存不会超过4GB这一特性,无论设置的是什么,Redis都把最大可用内存字节数控制在了3GB且默认选取了`REDIS_MAXMEMORY_NO_EVICTION`这一内存驱逐策略

以上讨论所涉及相关代码片段如下:

```c
//redis.h
struct redisServer {
    ...
    unsigned long long maxmemory;   /* Max number of memory bytes to use */
    int maxmemory_policy;           /* Policy for key eviction */
}

#define REDIS_DEFAULT_MAXMEMORY 0

//内存驱逐策略
#define REDIS_MAXMEMORY_VOLATILE_LRU 0
#define REDIS_MAXMEMORY_VOLATILE_TTL 1
#define REDIS_MAXMEMORY_VOLATILE_RANDOM 2
#define REDIS_MAXMEMORY_ALLKEYS_LRU 3
#define REDIS_MAXMEMORY_ALLKEYS_RANDOM 4
#define REDIS_MAXMEMORY_NO_EVICTION 5
#define REDIS_DEFAULT_MAXMEMORY_POLICY REDIS_MAXMEMORY_NO_EVICTION

//redis.c
struct redisServer server; /* server global state */

void initServerConfig() {
    server.arch_bits = (sizeof(long) == 8) ? 64 : 32; //32 or 64位架构
    server.maxmemory = REDIS_DEFAULT_MAXMEMORY;
    server.maxmemory_policy = REDIS_DEFAULT_MAXMEMORY_POLICY; //默认不驱逐
    ...
    //针对32位机器内存不会超过4GB这一特性,无论设置的是什么,Redis都把最大可用内存字节数控制在了3GB且默认选取了`REDIS_MAXMEMORY_NO_EVICTION`这一内存驱逐策略
    if (server.arch_bits == 32 && server.maxmemory == 0) {
        redisLog(REDIS_WARNING,"Warning: 32 bit instance detected but no memory limit set. Setting 3 GB maxmemory limit with 'noeviction' policy now.");
        server.maxmemory = 3072LL*(1024*1024); /* 3 GB */
        server.maxmemory_policy = REDIS_MAXMEMORY_NO_EVICTION;
    }
}
```

### maxmemory-policy

`maxmemory-policy`定义了内存在达到`maxmemory`规定的字节数以后所使用的key清理策略, 默认是不驱逐(`noeviction`)。有如下六种策略：

- volatile-lru

remove the key with an expire set using an LRU algorithm

对满足LRU算法且设置了过期时间的key进行清理

- allkeys-lru

remove any key accordingly to the LRU algorithm

对满足LRU算法的任意key进行清理

- volatile-random

remove a random key with an expire set

对设置了过期的key任意清理key

- allkeys-random

remove a random key, any key

任意key随机清理

- volatile-ttl

remove the key with the nearest expire time (minor TTL)

按照剩余过期时间从小到大排序依次清理

- noeviction

don't expire at all, just return an error on write operations

不清理内存，仅仅对写操作报错

## LRU

[LRU](https://en.wikipedia.org/wiki/Cache_replacement_policies#Least_recently_used_(LRU))(Least Recently Used)：最近最少使用策略，言外之意就是最近比较少使用的优先淘汰。

### 思路

```c++
class LRUCache {
public:
    int _capacity;
    unsigned int _size;
    unordered_map<int, list<pair<int, int>>::iterator> um; //key => node postion
    list<pair<int, int>> l; //node => <key,value>

public:
    LRUCache(int capacity) {
        this->_capacity = capacity;
        this->_size = 0;
    }

    int get(int key) {
        //查找key所在的位置
        //返回key的value
        //找不到
        auto mapKeyPos = this->um.find(key);
        if (mapKeyPos == this->um.end()) {
            return -1;
        }

        //能找到
        this->l.emplace_front(make_pair(mapKeyPos->second->first, mapKeyPos->second->second)); //插入链表头结点
        this->l.erase(mapKeyPos->second);
        this->um[key] = this->l.begin(); //更新散列表位
        return mapKeyPos->second->second;
    }

    void put(int key, int value) {
        if (this->_size >= this->_capacity) { //当前长度大于等于容量
            //删除链表
            int mapKey = (this->l.rbegin()->first); //要淘汰的key
            auto mapKeyPos = this->um.find(mapKey);
            this->um.erase(mapKeyPos); //删除散列表
            this->l.pop_back(); //删除链表尾结点
        } else { //当前长度尚未达到最大容量
            auto mapKeyPos = this->um.find(key);
            if (mapKeyPos != this->um.end()) { //能找到删除结点
                this->l.erase(mapKeyPos->second);
            }
            this->_size++;
        }
        this->l.emplace_front(make_pair(key, value)); //插入链表头结点
        this->um[key] = this->l.begin(); //更新散列表位
    }
};
```

无论是set还是put操作，都将所操作的key插入到链表的头节点，并删除对应的原节点，更新在散列表里的位置