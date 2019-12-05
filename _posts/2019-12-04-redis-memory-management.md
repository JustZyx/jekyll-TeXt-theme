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

- 涉及到频繁的内存迁移，插入删除操作较多，用链表来维护LRU结构
- 针对链表查找O(N)的时间复杂度劣势，引入散列表来维护key在链表中的位置

时间复杂度: O(1)

```c++
class LRUCache
{
private:
    list<pair<int, int>> l; //node => <key,value>
    int size; //当前长度
    int cap; //可容纳的容量
    unordered_map<int, list<pair<int, int>>::iterator> um; // <key, key在链表中迭代器位置>

public:
    LRUCache(int capacity) {
        this->cap = capacity;
        this->size = 0;
    }

    int get(int key) {
        auto umPos = this->um.find(key);
        if (umPos == this->um.end()) {
            return -1;
        }

        int value = umPos->second->second;

        this->l.erase(umPos->second);
        this->l.push_front(make_pair(key, value));
        this->um.erase(umPos);
        this->um[key] = this->l.begin();

        return value;
    }

    void put(int key, int value) {
        //先搜索, 如果找到删除
        //如果没找到, 判断是否size >= cap，如果reached，则删除末尾结点
        auto umPos = this->um.find(key);
        if (umPos != this->um.end()) {
            this->l.erase(umPos->second);
            this->um.erase(umPos);
        } else {
            if (this->size >= this->cap) {
                this->um.erase(this->l.rbegin()->first);
                this->l.pop_back();
            } else {
                this->size++;
            }
        }

        //新值插入头结点且更新在散列表中的位置
        this->l.push_front(make_pair(key, value));
        this->um[key] = this->l.begin();
    }
};

int main()
{
    auto cache = new LRUCache(2);

    cout << cache->get(2) << endl;
    cache->put(2, 6);
    cout << cache->get(1) << endl;
    cache->put(1, 5);
    cache->put(1, 2);
    cout << cache->get(1) << endl;
    cout << cache->get(2) << endl;

    return 0;
}
```

个人觉得这道题并不能算上Medium的难度，写完以后打脸了，这道题并不是难在思路，难在代码量上，我特意用秒表计了个时，标准的50行代码20分钟内白板上bugfree，基于这个背景确实很符合Medium难度。

### Redis LRU的实现
