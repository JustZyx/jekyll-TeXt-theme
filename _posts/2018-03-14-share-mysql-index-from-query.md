---
title: 再谈MySQL索引
key: 10005
tags: MySQL
layout: post
category: blog
comment: true
date: 2018-03-14 10:25:00 +08:00
modify_date: 2018-03-14 10:25:00 +08:00
---

## 前言
MySQL索引是一个老生常谈的话题，在开始本文之前希望大家都问自己几个问题：

* 什么是索引？
* 为什么索引可以加快查询速度？
* 索引是不是越多越好，为什么？
* 什么时候该建索引，什么时候不该建，判断的依据是什么？
* 如何评价一条sql实际的执行效果？
* b树和b+树的区别是啥？为什么最终选择了b+树这种数据结构？


如果您基础扎实，工程经验丰富，对上述几个问题非常清晰，那么希望老师傅帮忙Review一下本文，感激不尽。反之，我相信下面的内容应该多多少少能够给大家一点帮助。本文依托贷上钱业务库实际的例子将从以下三个方面来介绍一下MySQL的索引，力求深入浅出，老少咸宜：

* 索引的原理
* 慢查询优化
* 贷上钱业务库的慢查以及改进方案

注：1.由于MySQL版本变化、实际数据量以及业务的特殊性等因素，本文所提及的所有建议及实操方案均有其特殊性，切不可生搬硬套，需具体问题具体分析。本文所举例子并不能涵盖所有case，但对培养定位慢查的思维过程有一定的以不变应万变效果。2. 本文默认使用InnoDB存储引擎

## 索引的原理
1. 我们都知道，MySQL能支持千万级，甚至亿级别的数据量，这么大的数据肯定不可能放在内存中，那么只能是在磁盘上了。
2. 另外，我们也知道磁盘的读写开销远比内存读写开销大的多的多，如果是随机访问的话大约是10万倍级别的差距。
我们考虑一个问题，假设在10000个整数中查找某个整数，我们通常的做法是通过二叉搜索树，其平均复杂度是O(lgN)，时间复杂度已经相当优化，但这里我们忽略了一个关键的问题，复杂度模型是基于每次相同的操作成本来考虑的，数据库实现比较复杂，数据保存在磁盘上，而为了提高性能，每次又可以把部分数据读入内存来计算，因为我们知道访问磁盘的成本大概是访问内存的十万倍左右，所以简单的搜索树难以满足复杂的应用场景。

由此我们可以得出一个结论，**查找速度取决于磁盘IO次数，而磁盘IO次数又取决于树的高度**。那么为了满足高速查找的需求，我们是否可以构造一种高度可控的多路搜索树呢？这就是B+树。

### 简述B+树运行过程
![图片](https://lexiangla.com/assets/31166e06273311e8863b5254004b6d18 "图片")
浅蓝色的块我们称之为一个磁盘块，可以看到每个磁盘块包含几个数据项（深蓝色所示）和指针（黄色所示），如磁盘块1包含数据项17和35，包含指针P1、P2、P3，P1表示小于17的磁盘块，P2表示在17和35之间的磁盘块，P3表示大于35的磁盘块。真实的数据存在于叶子节点即3、5、9、10、13、15、28、29、36、60、75、79、90、99。非叶子节点只不存储真实的数据，只存储指引搜索方向的数据项，如17、35并不真实存在于数据表中。

如果要查找数据项29，那么首先会把磁盘块1由磁盘加载到内存，此时发生一次IO，在内存中用二分查找确定29在17和35之间，锁定磁盘块1的P2指针，内存时间因为非常短（相比磁盘的IO）可以忽略不计，通过磁盘块1的P2指针的磁盘地址把磁盘块3由磁盘加载到内存，发生第二次IO，29在26和30之间，锁定磁盘块3的P2指针，通过指针加载磁盘块8到内存，发生第三次IO，同时内存中做二分查找找到29，结束查询，总计三次IO。真实的情况是，3层的b+树可以表示上百万的数据，如果上百万的数据查找只需要三次IO，性能提高将是巨大的，如果没有索引，每个数据项都要发生一次IO，那么总共需要百万次的IO，显然成本非常非常高。

### 性质
1. 通过上面的分析，我们知道IO次数取决于b+数的高度h，假设当前数据表的数据为N，每个磁盘块的数据项的数量是m，则有h=㏒(m+1)N，当数据量N一定的情况下，m越大，h越小；而m = 磁盘块的大小 / 数据项的大小，磁盘块的大小也就是一个数据页的大小，是固定的，如果数据项占的空间越小，数据项的数量越多，树的高度越低。这就是为什么每个数据项，即索引字段要尽量的小，比如int占4字节，要比bigint8字节少一半。这也是为什么b+树要求把真实的数据放到叶子节点而不是内层节点，一旦放到内层节点，磁盘块的数据项会大幅度下降，导致树增高。当数据项等于1时将会退化成线性表。

2. 当b+树的数据项是复合的数据结构，比如(name,age,sex)的时候，b+数是按照从左到右的顺序来建立搜索树的，比如当(张三,20,F)这样的数据来检索的时候，b+树会优先比较name来确定下一步的所搜方向，如果name相同再依次比较age和sex，最后得到检索的数据；但当(20,F)这样的没有name的数据来的时候，b+树就不知道下一步该查哪个节点，因为建立搜索树的时候name就是第一个比较因子，必须要先根据name来搜索才能知道下一步去哪里查询。比如当(张三,F)这样的数据来检索时，b+树可以用name来指定搜索方向，但下一个字段age的缺失，所以只能把名字等于张三的数据都找到，然后再匹配性别是F的数据了， 这个是非常重要的性质，即索引的最左匹配特性。

## 慢查询优化
提到慢查，当我还是小白的时候，老师傅一般都是粗暴的让我怼一个索引上去。但是如何判断一个索引应不应该加？如何评价索引的效果？

1.  **尽可能**选择区分度高的列作为索引，区分度公式是 
```
count(distinct(column)/count(*)
```
表示字段不重复的比例，取值范围是(0,1]，唯一键的区分度为1，enum，tinyint等则区分度无限趋近于0。区分度越高扫描的记录数越少，反之越多，索引代价较高，一般需要join的字段索引区分度建议0.1以上，即平均1条扫描10条记录

2.  提到索引效果就不得不提查询优化神器 — [EXPLAIN](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html)命令了。
其中rows是核心指标，绝大部分rows小的语句执行一定很快（有例外，下面会讲到）。所以优化语句基本上都是在优化rows。

3. 最左前缀匹配原则
mysql会一直向右匹配直到遇到范围查询(>、<、between、like)就停止匹配，比如a = 1 and b = 2 and c > 3 and d = 4 如果建立(a,b,c,d)顺序的索引，d是用不到索引的，如果建立(a,b,d,c)的索引则都可以用到，a,b,d的顺序可以任意调整

4. 实在无法优化的只读查询，祸水东引至从库。

## 贷上钱业务实例
先简单介绍一下慢查所涉及的相关表结构
* 订单表(apply)
![图片](/assets/images/blog/5f165fd821ba11e88a4b5254004fae61 "图片")
* 支付流水表(pay_log)
```
CREATE TABLE `pay_log` (
  `pay_log_id` int(11) NOT NULL AUTO_INCREMENT COMMENT '自增id',
  `pay_log_serial_number` varchar(64) NOT NULL DEFAULT '' COMMENT '交易流水号',
  `pay_log_biz_order_number` varchar(128) NOT NULL DEFAULT '' COMMENT 'BIZ订单号',
  `pay_log_type` varchar(16) NOT NULL DEFAULT '' COMMENT '关联的支付类型  transaction-还款',
  `pay_log_target_id` int(11) NOT NULL DEFAULT '0' COMMENT '关联的记录id',
  `pay_log_id_card` char(18) NOT NULL DEFAULT '' COMMENT '身份证号',
  `pay_log_amount` decimal(10,2) NOT NULL DEFAULT '0.00' COMMENT '支付金额（单位：分）',
  `pay_log_create_at` datetime NOT NULL DEFAULT '1000-01-01 00:00:00' COMMENT '创建时间',
  `pay_log_finish_at` datetime NOT NULL DEFAULT '1000-01-01 00:00:00' COMMENT '成功时间',
  `pay_log_status` tinyint(1) NOT NULL DEFAULT '0' COMMENT '支付状态 0-未知 1-成功 2-失败',
  `pay_log_message` varchar(512) NOT NULL DEFAULT '' COMMENT '交易信息',
  `pay_log_push_biz_status` tinyint(1) NOT NULL DEFAULT '0' COMMENT '通知biz状态 0-未知 1-成功 2-失败',
  `pay_log_repay_type` tinyint(2) NOT NULL DEFAULT '0' COMMENT '还款方式 0-主动还款，1-代扣还款',
  PRIMARY KEY (`pay_log_id`),
  UNIQUE KEY `pay_log_idx1` (`pay_log_serial_number`),
  KEY `pay_log_idx2` (`pay_log_type`,`pay_log_target_id`,`pay_log_status`),
  KEY `pay_log_idx3` (`pay_log_id_card`,`pay_log_status`,`pay_log_push_biz_status`),
  KEY `pay_log_idx4` (`pay_log_biz_order_number`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='支付记录表';
```

1. 明确应用场景
先来看长期以来占据慢查排行榜之首的一条慢查(慢查总量百分之六七十这样)
```
SELECT * 
FROM `pay_log` 
WHERE (`pay_log_status`=0) 
	AND (`pay_log_type` IN ('transaction-biz', 'alipay', 'weixinpay', 'deferment-biz', 'shortlink')) 
	AND (`pay_log_biz_order_number` != '0');
```
 1.1 执行时间
 1.2 EXPLAIN (生产环境以优化，以测试环境举例)
 ![图片](/assets/images/blog/96e09cb0273411e8a1955254004b6d18 "图片")
 1.3 分析
 如果有细心的同学应该会发现，第一条建议尽可能三个字被我加粗了，本例子就是那条建议的一个例外。按照区分度公式，pay_log_status字段显然是不适合加索引的，明显区分度太低了。但是通过了解业务发现pay_log_status只是一个中间态，业务方每隔几分钟就会从biz异步来更新成1或者2，所以几分钟内的数据量不会太大。而pay_log_type就不一样了，pay_log_type的数据分布较为均匀，加索引也无法锁定特别少量的数据。
 ![图片](/assets/images/blog/43acfeca273511e89eb55254009d059e "图片")
 ![图片](/assets/images/blog/6810906a273511e88c2f5254009d059e "图片")
由此，我删除了之前的
```
KEY `pay_log_idx2` (`pay_log_type`,`pay_log_target_id`,`pay_log_status`)
```
新建了一个status的索引
```
KEY `pay_log_idx2` (`pay_log_status`,`pay_log_type`)
```
1.4 效果
![图片](/assets/images/blog/441e64ec273611e89d435254009d059e "图片")


2. offset过大的问题
**慢查如下**
```
# Time: 180308 16:00:02
# User@Host: paydayloan[paydayloan] @  [10.105.99.4]  Id: 1719727201
# Query_time: 20.728928  Lock_time: 0.000085 Rows_sent: 0  Rows_examined: 29454299
SET timestamp=1520496002;
SELECT `apply`.*
FROM `apply`
LEFT JOIN `oauth_user` ON apply_oauth_user_id = oauth_user_id LEFT JOIN `individual` ON individual_id = oauth_user_individual_id WHERE (`apply_status`='wait_credit') 
	AND (`apply_create_at` >= '2018-02-01 00:00:00') 
	AND (`apply_create_at` <= '2018-02-28 00:00:00') 
	ORDER BY `apply_id` DESC 
	LIMIT 10 
	OFFSET 5290;
```
**改写之后**
```
select *
from `apply` as a
inner join (
	SELECT `apply`.apply_id 
	FROM `apply` 
	LEFT JOIN `oauth_user` ON apply_oauth_user_id = oauth_user_id 
	LEFT JOIN `individual` ON individual_id = oauth_user_individual_id 
	WHERE 	(`apply_status`='wait_credit') 
	AND (`apply_create_at` >= '2018-02-01 00:00:00') 
	AND (`apply_create_at` <= '2018-02-28 00:00:00')
) as b
	on a.`apply_id` = b.`apply_id`
	ORDER BY b.`apply_id` DESC LIMIT 10 OFFSET 5290;
```

本文最初由本人在公司内部分享之用，现已征的公司同意，可以公开.  By Yongxin.Zhaung