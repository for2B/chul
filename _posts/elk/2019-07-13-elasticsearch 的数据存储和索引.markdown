---
layout:     post
title:      "elasticsearch 的数据存储和索引"
subtitle:   "es学习记录"
date:       2019-07-13
author:     "CHuiL"
header-img: "img/elk-bg.png"
tags:
    - elk
---


### 倒排索引
elasticsearch底层是使用Lucene的倒排索引技术来实现比关系型数据库更快的过滤，在了解什么是倒排索引之前，先明白几个关键词的意思。



###### index：索引，在倒排索引里一个index就相当于一个数据库。
###### document：文档，一个文档归属一个索引下面，有自己的docId。相当于一条数据。
###### field：字段，一个文档由多个字段组成。
###### term：字段的内容，就是某个field的值，大部分情况下都有多个值。
###### posting list: docId的集合。
##### 例子
docid | 年龄 |  性别 |
---|---|---|
1 | 18| 女|
2 | 20 | 女|
3 | 18 | 男|

假设我有一个索引为person,而上述表格中每一行就是代表一个文档，field就是年龄，性别。而年龄的term就是18 20，性别就是男女。则上述关系存在如下的倒排索引
###### 年龄
term | posting list 
---|---
18 | [1,3]
20 | [2]
###### 性别
term | posting list 
---|---
女 | [1,2]
男 | [3]

所以不同于我们平时在关系型数据库里使用的索引，比如为id建立一个索引，然后通过b+tree去加快搜索。当我们需要根据年龄的具体数值去查询某一部分数据时，正向索引是根据索引遍历得到结果（Btree的话就是二分查找得到）；而倒排索引则是根据term到postring list的映射，直接获得关键字对应的文档列表。当经的搜索引擎检索算法也是基于倒排索引来实现的。  
所以倒排索在多条件的过滤和组合查询上性能要高过正向索引。    
  
#### 根据倒排索引检索
假设我们某个field上有很多值，比如姓名。如果每个姓名都不重复，且是乱序的，那么根据名字这个关键字去检索也还是需要全部遍历一遍。所以term还是需要排序的，排好序之后便可以使用二分查找加快排序速度。而排序后的term就是==term dictionary==。

但是这样的话，不就跟普通数据库的索引查询一样了吗？数据库索引是通过将数据组织成b-tree的形式存放在物理盘上以减少磁盘的读写次数来提高检索性能，而es则是希望直接通过内存来查找term，不读磁盘。但是正如前面的名字feld，如果有很多条数据，每条都不重复，那么内存肯定不够放。于是es组织一棵树，这颗树不包含所有的term，它包含term的一些前缀到block之间的映射关系，在结合FST的压缩技术，使它能够存放到内存中，称为==term index==。  
  
顾名思义，就是为了快速查找到term的索引。在内存中快速定位term的前缀所在block后，在去磁盘顺序查找term，大大减少了磁盘的读取次数。  

![image](https://ask.qcloudimg.com/http-save/yehe-1729674/18lqv5kq8c.png?imageView2/2/w/1620)

#### 倒排索引-合并查询
先略

#### Mapping
先略
#### Segment文件
一个Segment文件内部存储着倒排索引，所以一个Segment文件即是一个完备的lucene倒排索引。Segment文件是现在内存中生成的，然后再经过一个==refresh==时间间隔（默认1s，可通过refresh_interval参数设置，需查看官方文档）后就会将数据写入文件系统缓冲区中，这样便可以对新进的数据进行缓冲。注意，此时仅仅只是写入到缓冲中，换句话说，一但重启数据将丢失。  
 
为了保证数据不丢失，再将数据写入内存中时还会维护一份文件==translog==，translog也是一开始在内存记录，每隔五秒写一次磁盘，且在每次 index、bulk、delete、update 完成的时候，一定触发刷新 translog 到磁盘上，才给请求返回 200 OK。一旦重启服务器对于还没罗盘的数据便可以通过translog来进行恢复。  
    
在经过一个==flush==时间间隔（默认30分钟,`index.translog.flush_threshold_period`），或者当translog文件大小大于一定大小时（512MB,`index.translog.flush_threshold_size`），就会执行flush操作，将数据同步到磁盘。一旦写入磁盘，segment文件便不在修改，就算时update操作，也是先delete后重新创建。而后translog文件会保留一段时间，默认保留12小时。大小512m。

由于这种机制会导致短时间内生成很多个segment文件，而文件是要占用文件描述符，而且每次搜索时都要从这么多个segment文件中去搜索，显然性能会非常底。所以es会在后台对segment做一个合并的操作，这个过程是有独立的线程来进行的。 合并完成后小的segment文件就会被删除。 
  
归并线程是按照一定的运行策略来挑选 segment 进行归并的。主要有以下几条：

> index.merge.policy.floor_segment 默认 2MB，小于这个大小的 segment，优先被归并。
> index.merge.policy.max_merge_at_once 默认一次最多归并 10 个 segment
> index.merge.policy.max_merge_at_once_explicit 默认 forcemerge 时一次最多归并 30 个 segment。
> index.merge.policy.max_merged_segment 默认 5 GB，大于这个大小的 segment，不用参与归并。forcemerge 除外。

每个segment包含所有字段的倒排索引。 其中.tim文件包含所有字段的term dictionary; 另外有一个.tip后缀的文件存放所有字段term dictionary的索引，通过这个文件可以先定位到某个字段的term dictionary索引(FST编码)存放的位置，然后通过这个索引快速查找到该字段的某个term在.tim文件里的存放位置。

而常驻在heap的数据，是用于快速访问segment文件的一种索引结构，segment本身不会常驻heap，只有被搜索访问到的文件块会被os page cache缓存。

#### 分片和索引 
一个索引可以设置多个分片。每个分片包含多个segment；  
查询的时候，会把所有的segment查询结果汇总并作为最终分片查询结果返回；  
在分布式集群中，不同分片会位于不同的节点上，当想某一个节点发出请求时，会简单根据文档id哈希求模得到该文档所在的分片，进而路由到该分片所在的node，如果进行增删改的操作，在分片执行成功之后会并行发送请求到其他的副本分片上，只有等到报告成功之后才会向原来接受请求的节点报告成功，并由原来的节点返回给客户。







### 索引的使用
https://www.cnblogs.com/linjiqin/p/10764101.html

### 索引原理
https://cloud.tencent.com/developer/article/1368187
https://www.cnblogs.com/dreamroute/p/8484457.html
https://elasticsearch.cn/article/6178  


### segment
https://elasticsearch.cn/article/6178  
https://www.jianshu.com/p/9b872a41d5bb  
https://www.ctolib.com/docs/sfile/ELKstack-guide-cn/elasticsearch/principle/realtime.html  
https://www.cnblogs.com/richaaaard/p/5226334.html  
https://elasticsearch.cn/article/32
