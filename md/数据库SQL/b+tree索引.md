磁盘地址  id

文件系统【磁盘，柱面，磁道，扇区】

操作系统 时间局部性，空间局部性     Page

#### 什么是索引

索引时帮助高效获取数据的数据结构

索引也可能是一个文件

其他类型：

> Hash   Map
>
> 二叉树
>
> 红黑树

### 为什么用 B+ 树实现

* engine

innodb

>idb，以主键为索引来组织数据，叶子存数据
>
>副索引 存 主键
>
>聚集索引

![](https://ws1.sinaimg.cn/large/006xzusPgy1g23oorbic8j30r30dfn1a.jpg)

myisam

>myi文件，索引 B+ Tree  ，叶子存 myd 地址-，副索引也存地址>
>
>myd文件，表
>
>非聚集索引

![](https://ws1.sinaimg.cn/large/006xzusPgy1g23om06ta0j30tg0drgqu.jpg)

![](https://ws1.sinaimg.cn/large/006xzusPgy1g23oqbv12hj30ss0bt0wb.jpg)

B树，每个节点（key,data）二元组

B+树，叶子节点



depth，node 多数据

### 为什么用自增 id 作为主键？

CRUD

* 自增 id vs UUID

相对于 B+树

自增 id 数据插入总在最右边，（一个一个Page填满），数字比较

UUID 随机插入数据，数据分裂/移动相对频繁，字符串比较





![](https://ws1.sinaimg.cn/large/006xzusPgy1g23omx5jh4j30a6079q3w.jpg)



列的离散型：

比例越高，离散性越大

离散性越高，选择性就越好



最左匹配原则

对索引中关键字进行计算（对比），一定是从左往有依次进行，且不可跳过



### 联合索引

单列索引是特殊的联合索引

联合索引列选择原则

1. 经常使用的列优先【最左匹配原则】
2. 离散度越高的列优先【离散度高原则】
3. 宽度小的列优先【最小空间原则】





### 覆盖索引

如果查询的列，通过索引项的信息可直接返回，则该索引称之为查询SQL的覆盖索引





















snowflake     long  数字