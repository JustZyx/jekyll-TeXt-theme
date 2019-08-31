---
title: Opcache踩过的一些小坑
key: 10006
tags: opcache
layout: article
category: blog
comment: true
date: 2017-09-18 16:24:00 +08:00
modify_date: 2017-09-18 16:24:00 +08:00
---

## 背景
贷上钱在6月份的时候由于运营的批量召回使得应用服务器CPU飙升至满载，导致贷上钱APP有几个小时不能正常提供服务，当时的应急方案是通过额外加了两台应用服务器临时缓解了压力。虽然横向扩展能够解决，但显然不是最佳方案。最终通过blackfire性能分析工作定位出CPU消耗所在，然后做了两处小改动，其中一点就是开启了PHP7 Zend引擎自带的Opcache扩展。启用扩展之后效果很明显，可以从腾讯云后台很明显的看到CPU占用降低了大约百分之40左右。虽然我很早之前就用过Opcache扩展，但是还是遇到了一个困惑很久的问题。冯少建议发出来和大家一起交流一下，公司有很多大佬，如果有说的不对的地方希望及时指正。

## 要点
1. 发布完代码之后，部分请求走不到最新的release目录。
2. Opcache建议的配置（主要针对缓存更新时机）


## 描述
我们都知道PHP是弱类型语言，底层采用Zend引擎+扩展的模式降低内部耦合，Zend引擎是C语言实现的，也称为PHP的内核，它将PHP的代码通过一系列操作（词法分析，语法分析等编译过程）翻译成opcode。这一步操作是最耗CPU的过程。opcache扩展就是为了解决这个问题，对同一段代码完全没必要每次都重新解析成opcode，尤其在QPS较高的时候，可以大量降低不必要的CPU消耗。有一本关于PHP内核方面的电子书感觉还不错，有兴趣的可以看一下：http://www.php-internals.com/book/。
	
从PHP 5.5起，Opcache扩展自带的，但是默认关闭。谨慎起见，在其中的一台应用服务器开启了Opcache扩展。在观察了几天之后有同事反映说在sentry发现有请求走到了老的release目录。第一时间当然怀疑是opcache的问题，可难处在于由于是生产环境，无法进行多次测试，所以第一次粗暴的进行了fpm进程重启之后就好了。但是第二次版本发布的时候又出现了类似的情况，无奈之下，只好先把扩展拿掉了。上线之前在测试环境中做过相关的opcode缓存更新的测试，加上是由于走错路经，而不是走到正确的代码，代码没生效，所以可以排除并不是opcode的缓存问题。一开始的关注点是server，理由是寻址是server寻的，但是没装opcache扩展的机器为什么server没有寻址错误呢？那么只有一种解释，fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;被缓存了，查看opcache的官方文档果然有一个选项， opcache.revalidate_path。
	
这是因为PHP中有两种cache，一种是opcode cache，一种是realpath cache（http://jpauli.github.io/2014/06/30/realpath-cache.html）。Zend引擎下opcode cache 是通过 realpath cache 获取文件信息，即便软链接已经指向了新位置，但是如果 realpath cache 里还保存着旧数据的话，opcode cache 依然无法知道新代码的存在，缺省情况下opcache.revalidate_path是关闭的，此时会缓存符号链接的值，这会导致即便软链接指向修改了，也无法生效。还有一种可行方案是从server层面来解决，每个请求都会经过server，nginx可以通过配置$realpath_root来强制每次都走真实的路径(http://nginx.org/en/docs/http/ngx_http_core_module.html#var_realpath_root)，但是这会带来一个副作用，会造成额外的IO开销，性能会有略微下降。


## 建议
1. opcache建议配置

    opcache.enable=1

    opcache.enable_cli=1

    opcache.revalidate_path=1	//路径检测

    opcache.memory_consumption=128

    opcache.max_accelerated_files=2000

    opcache.interned_strings_buffer=8

2. 在发布系统post_release中执行opcache_reset();以替代opcache的文件变动检测opcache.revalidate_freq=1,opcache.validate_timestamps=1,主要是因为文件变动检测会造成额外的系统开销，而这种开销完全是没有必要的，在没有发布新代码之前理论上代码是不可能发生变动的。所以只需要发布完代码之后进行一次重制opcode缓存的操作即可。

3. Yii2框架配置文件里不要为了用一个常量去use某个Model，全局常量应尽可能的通过require或use一个常量文件。blackfire跟踪下来发现，use Model会造成百分之一的额外CPU开销。

## 总结
1. Nginx官方文档和PECL帮了大忙
2. 时刻保持谨慎
