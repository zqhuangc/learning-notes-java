[数据结构与对象的关联](https://yq.aliyun.com/articles/552853)

## 前言

Redis用到的底层数据结构有：简单动态字符串、双端链表、字典、压缩列表、整数集合、跳跃表等，Redis并没有直接使用这些数据结构来实现键值对数据库，而是基于这些数据结构创建了一个对象系统，这个系统包括字符串对象、列表对象、哈希对象、集合对象和有序结合对象共5种类型的对象

## 底层数据结构

###  简单动态字符串（sds）

redis自定义了简单动态字符串数据结构（sds），并将其作为默认字符串表示。



###  链表（）

- 链表被广泛用于实现Redis的各种功能，比如列表键、发布与订阅、慢查询、监视器等。
- 每个链表节点由一个listNode结构表示，每个节点都有一个指向前置节点和后置节点的指针，所以Redis中链表是双向链表。

###  字典 （hashtable）

字典被广泛用于实现Redis的各种功能

- Redis中字典使用哈希表作为底层实现，每个字典有2个哈希表，一个平时使用，另一个只在rehash时使用。
- 当字典作为数据库的底层实现，或者作为哈希键的底层实现时，使用MurmurHash2算法计算键的哈希值。

#### rehash

 Redis的dictEntry也是通过链表（next指针）方式来解决hash冲突

 

 Redis在满容有驱逐策略的情况下，Master/Slave 均有大量的Key驱逐淘汰，导致Master/Slave 主从不一致



Slave内存区域比Master少一个repl-backlog buffer（线上一般配置为128M）

 

在rehash 进行期间，每次对字典执行CRUD操作时，程序除了执行指定的操作以外，还会将ht[0]中的数据rehash 到ht[1]表中，并且将rehashidx加一

 **rehashidx是下一个需要rehash的项在ht[0]中的索引(对应bucket)，不需要rehash时置为-1。也就是说-1时，表示不进行rehash**





_dictRehashStep中，也会调用dictRehash，而_dictRehashStep每次仅会rehash一个bucket从ht[0]到 ht[1]

```ANSI C
typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    int rehashidx; /* rehashing not in progress if rehashidx == -1 */
    int iterators; /* number of iterators currently running */
} dict;


/* 将Key插入哈希表 */
dictEntry *dictAddRaw(dict *d, void *key) 
{ 
    int index; 
    dictEntry *entry; 
    dictht *ht; 

    if (dictIsRehashing(d)) _dictRehashStep(d);  // 如果哈希表在rehashing，则执行单步rehash

    /* 调用_dictKeyIndex() 检查键是否存在，如果存在则返回NULL */ 
    if ((index = _dictKeyIndex(d, key)) == -1) 
        return NULL; 

    // rehash 时数据插入会去到 ht[1]
    ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0]; 
    entry = zmalloc(sizeof(*entry));   // 为新增的节点分配内存
    // 头插法
    entry->next = ht->table[index];  //  将节点插入链表表头
    ht->table[index] = entry;   // 更新节点和桶信息
    ht->used++;    //  更新ht

    /* 设置新节点的键 */ 
    dictSetKey(d, entry, key); 
    return entry; 
}



int dictRehash(dict *d, int n) {
	if (!dictIsRehashing(d)) return 0;
	while (n--) { 
		dictEntry *de, *nextde;
		if (d->ht[0].used == 0) {   // 如果 0 号哈希表为空，那么表示 rehash 执行完毕
			zfree(d->ht[0].table);
			d->ht[0] = d->ht[1];
			_dictReset(&d->ht[1]);
			d->rehashidx = -1;
			return 0;
		}
 
		// Note that rehashidx can't overflow as we are sure there are more
		// elements because ht[0].used != 0
		// 确保 rehashidx 没有越界
		assert(d->ht[0].size > (unsigned)d->rehashidx);
 
		while (d->ht[0].table[d->rehashidx] == NULL) d->rehashidx++;    // 略过数组中为空的索引，找到下一个非空索引
 
		de = d->ht[0].table[d->rehashidx];
		while (de) {
			unsigned int h;
			nextde = de->next;
			// 计算新哈希表的哈希值，以及节点插入的索引位置
			h = dictHashKey(d, de->key) & d->ht[1].sizemask;
			// 插入节点到新哈希表,而且是插入到表头
			de->next = d->ht[1].table[h];
			d->ht[1].table[h] = de;
 
			d->ht[0].used--;
			d->ht[1].used++;
			de = nextde;
		}
		// 将刚迁移完的哈希表ht[0]对应索引的指针设为空
		d->ht[0].table[d->rehashidx] = NULL;
		d->rehashidx++;
	}
	return 1;

```



### 跳跃表（skiplist）

 Redis中的跳跃表由zskiplistNode和zskiplist两个结构体定义，其中zskiplistNode表示跳跃表节点，zskiplist表示跳跃表信息。



 Redis使用跳跃表作为有序集合的底层实现之一，如果一个有序集合包含的元素数量较多，或者有序集合元素是比较长的字符串，Redis就会使用跳跃表作为有序集合的底层实现。

### 整数集合 （ intset ）

整数集合是集合键的底层实现之一，当一个集合只包含整数元素时，并且每个集合的元素数量不多时，Redis就会使用整数集合作为集合建的底层实现。

 

### 压缩列表（）

 压缩列表是列表键和哈希表键的底层实现之一，当一个列表键只包含少量列表项，并且每个列表项是小整数或者短的字符串，那么会使用压缩列表作为列表键的底层实现。

 

压缩列表是Redis为了节约内存开发的，由一系列特殊编码的连续内存块组成的顺序性数据结构。一个压缩列表可以包含多个节点，每个节点保存一个字节数组或者一个整数值

 

 

## Redis中的对象

Redis中共有5种不同类型的对象，分别是字符串、列表、哈希表、集合、有序集合。

 

 

 