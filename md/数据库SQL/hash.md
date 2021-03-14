来源：blog.csdn.net/liweisnake/article/details/104779497

- 关于java中hash的数据结构
- 关于hash的一些解决方案

------

最近对hash有了更多深入的理解。这里也写篇文章专门来聊聊hash。

Hash是一种常见的数据结构或者说计算方法，以其O(1)的时间算法复杂度闻名于世。曾有人说，如果世界上只有一种数据结构，那么我选择hash，足见hash的地位及牛逼之处，而代码编写中hash也屡见不鲜，因为他实在是太常见太好用了。

但是实际使用过程中，基本的hash是远远不够的，按照用途，对hash其实还有如下需求：

# 关于java中hash的数据结构：

1.**并发安全**。

对这个需求，java中有了HashTable，为了进一步提升性能，于是有了使用分段锁的ConcurrentHashMap，亦不做赘述。

2.**大数据hash**。

传统的HashMap中除了key, value外，每个entry还要存16个byte的class header，4byte的hash值，以及8byte的指向下一个元素的指针，这样的结构在遇到大数据量时就会更加耗内存，更容易导致GC。

![img](https://mmbiz.qpic.cn/mmbiz_png/JdLkEI9sZfdNbsKXtgzMj9FI5gJFFmvBCetgvPGIgTP51bkXkmXic86noaY90snmJtNZUoGSMTNUGScG1DyciaVg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

由对象头过大可以看出来，只要能够有一种结构消灭这个额外的entry对象，则此处将大大减少内存的消耗。

一种可行的方式是：采用二级索引保存的方式，第一级索引由Short2ShortMap保存一个short为key且short为value的Map结构，第二级索引则由许多数组构成，这些数组负责将被消灭value这个Object拆解为基本类型并用多个数组保存，而一级索引的value保存的value正是二级数组的index。通过这种变换，消灭了额外的entry对象从而大幅减少内存。需要注意的是，这种方式适用于使用了大量HashMap，但是每个Map内数据量较小的情况（受short的限制只有3w多index），如果每个Map内数据量也比较大，可以考虑Int2IntMap，当然，这样减少内存占用的效果就不如Short2ShortMap了。

![img](https://mmbiz.qpic.cn/mmbiz_png/JdLkEI9sZfdNbsKXtgzMj9FI5gJFFmvBwib5vnAXiaiaZ1yXxLM2UxD8Ev6H8bWVBG5mHse4Sp5NicZuddEMPibktFA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

3.**其他**。

ImmutableMap，Guava库，在初始化完毕后就没法再put做改变了。

SortedMap，Guava库，数据会按key做字母化排序。

BiMap，Guava库，创建完之后可以使用inverse将value和key颠倒过来，前提是保证value也是唯一的。

MultiMap，Guava库，可以对每个key关联多个值，并且可以很方便的对list进行分组。

# 关于hash的一些解决方案：

**Hash冲突**。众所周知，解决hash冲突最好的办法自然是提升hash table的总数量（即N的大小），如果待存放元素的数量k远小于N，则hash后有更大概率占据空槽，而冲突越少则性能越好，本质上，这是一种以空间换时间的方式。然而现实中，存储空间也很宝贵，任何公司都很难接受让大量空间浪费。于是，便出现了尽可能增加空间占用但不过分降低性能的hash。

**布谷hash**。布谷hash是一种解决冲突的方法。不同于线性探测，开放定址这样的常规方法，布谷hash借鉴了布谷鸟占人巢穴生子的寓意。其算法比较简单，采用两个（或多个）hash函数F1和F2，put操作时用F1或F2计算hashcode并定位，如果任意位置为空，则插入；否则挤占其中一个位置，并将被挤占的元素拿出并重复该过程；而get操作则让人比较困惑，到底采用哪个函数来get值呢？实际上布谷hash需要在value中存放key值，这样对于两个函数get到的值只要判断中间key是否正确就可以确认其对应的hash函数。布谷hash在二维时空间利用率较高，约为80%-90%。下图是对put操作的一个表示。

![img](https://mmbiz.qpic.cn/mmbiz_jpg/JdLkEI9sZfdNbsKXtgzMj9FI5gJFFmvBXULcDXwYuBtQ9hQJ9amutja6icBZmJw3GGRfVEUP0IIBV3HJB96XN3Q/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)cuckoo_preview

参考 https://coolshell.cn/articles/17225.html

**bloomfilter** 。布隆过滤器是一种占小空间且效率很高的算法，通常用来解决垃圾邮件识别，缓存击穿及日活计算等场景。bloomfilter只能判断一个元素可能在其中或者一个元素一定不在其中。他的算法也采用多个hash函数，如下例，某数据A经过x函数可以映射到4,9两个位置，经过z函数可以映射到9,14两个位置，经过y函数可以映射到14,19两个位置。于是基本的增加操作便可以将这几个对应位置的值置为1；对于基本的查找操作，则对A进行hash后找到其所有对应位置，发现其所有对应位置都是1，则表示A很可能存在，为什么不能确定呢，因为有可能这些位置并不是对A进行hash后对应的位置，有可能是插入了BCDE等数据而这些数据刚好覆盖了A的所有位置而导致的，所以发现全1仅仅能判断其可能存在；但是一旦有任意对应位置为0，则表示A一定不存在。对于基本的删除和更新操作，布隆过滤器是不支持的，本质原因是位置是多数据共享的，任何对数据的逆向操作都会导致其他数据的不准。布隆过滤器在Guava中有现成的实现。

![img](https://mmbiz.qpic.cn/mmbiz_png/JdLkEI9sZfdNbsKXtgzMj9FI5gJFFmvBGwJkN44W39TLThflYzt1qYfJ9vrwD8jtLrxTO3m1HQr5CyibUdHbqOw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

参考 https://juejin.im/post/5de1e37c5188256e8e43adfc

**\**Count–min sketch\**** 。Count-min sketch旨在解决流式大数据下做计数统计时间空间复杂度过高的问题。设想这样一个场景，线上数据源源不断的进来，现在我们需要去统计cache中每个ip请求的大致数量，从而确定哪个ip来的请求是hot的。碰到这个问题，可能本能的会想用HashMap，用ip作为key，用总count当做value。但实际上当数据量足够大时，各种噩梦就来了，比如每台机器内存非常高（对应上面说到的大数据hash带来的问题），hash冲突也变高，rehash成本也会迅速增加，并且在实时响应的要求下，时间上就很可能无法满足需求，Count-min sketch算法就是为此而生的。

count-min sketch算法思想比较简单，采用n个数组以及n个hash函数，对同一个数据用不同的hash函数做hash，分配到这n个数组不同的位置，存值时这个位置所在的value加1，取值时取这n个位置最小值，则此最小值大致接近实际总count数，且总是大于等于实际的总count数。为啥要取最小值，并且为啥结果总是大于等于实际总count数呢，原因其实与bloomfilter比较像，有可能有其他的hash也落到了该位置并加了count。参考下图。在java中，著名的caffeine缓存框架中的W-TinyLFU就用的Count-min sketch来记录访问频率

参考https://www.cnblogs.com/liujinhua306/p/9808500.html

![img](https://mmbiz.qpic.cn/mmbiz_png/JdLkEI9sZfdNbsKXtgzMj9FI5gJFFmvBh6moG5RXJ7z2gKutiaosnaO7vojQ4zaQZFZicDQgRC2BLtJDrL8v56yQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)query1

![img](https://mmbiz.qpic.cn/mmbiz_png/JdLkEI9sZfdNbsKXtgzMj9FI5gJFFmvBDicV9ZtLnrvyGcic9q6IwasYLo07icnicUek0umuIbmldy6gyyzecCZd3Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)query2

4.**hash分散**。大多数情况下，希望hash之后的结果越分散越无规律越好。

Murmur hash。Murmur哈希是一种比大多数算法更为分散更无规律的算法。

java中的hash算法称为Horner，简单表示就是

> for (int i = 0; i < str.length(); i++) { hash = 31*hash + str.charAt[i]; }

实际计算时经常使用移位操作。

Murmur的意思是multiply and rotate，主要优点是速度快且hash值足够分散，目前已经在各大框架广泛使用，比如redis，memcache，cassandra，lucene，如下是其简单表示。

> x *= m; x = rotate_left(x,r);

具体算法可参考 https://zh.wikipedia.org/zh-cn/Murmur%E5%93%88%E5%B8%8C

关于Murmur hash的科普参考这里 http://calvin1978.blogcn.com/articles/murmur.html

5.**hash聚集**。少数情况下，希望通过hash能让相似的内容在hash过后仍然相似，而不是一点改动便面目全非。

simhash。simhash是一种局部敏感hash，对于google百度这样的大搜索公司，用空间向量去计算文档相似度显得既慢又笨重，simhash用一种相似则海明距离近的方式巧妙而快速的解决了文档相似的比较。这对hash提出了另一种不同的要求，以往hash函数的目的是为了足够分散，而这里却希望hash后呈现一定的规律，实际上个人觉得这更像是一种编码，根据这种编码规则，相似的文档在hash值上的海明距离更近。

![img](https://mmbiz.qpic.cn/mmbiz_jpg/JdLkEI9sZfdNbsKXtgzMj9FI5gJFFmvBD5xHrvticlPX1w27P3AYVVBTUS5kCFQObPaHEDgRSibyQlANU2B5Psvg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

算法这里不再赘述，可参考 https://wizardforcel.gitbooks.io/the-art-of-programming-by-july/content/06.03.html

6.**其他特殊hash**。

**一致性hash**。一致性hash主要是为了解决传统的取模为主的hash将数据分配到n台服务器之后，服务器再扩容或缩容所带来的所有数据需要重新计算hash的问题。这种情况对于线上某些重要的服务往往是不可接受的。于是一致性hash出现了，它通过将hash值空间预先分配到一个超级大的虚拟节点上，再通过实体节点就近接管虚拟节点来解决映射问题。如图，这个超级大的虚拟节点即是2^32个，真正的的实体节点只有4个，由于顺时针就近映射，每个实体节点都将接管落入前面一个实体节点以后的所有虚拟节点的值，这样每次扩容时只会影响最多一个节点。一致性hash基本人尽皆知，这里就不列举资料了。

![img](https://mmbiz.qpic.cn/mmbiz_png/JdLkEI9sZfdNbsKXtgzMj9FI5gJFFmvBicdhVBqW6wTgz6UvyFELwh8URibOVCeeYskGGibYWe8cxXd8yo9AH3tEA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**Perfect hash**。perfect hash目的是为了实现完全无冲突的hash。perfect hash分为两种，一种是静态hash，一种是动态hash；对于静态hash而言，一个最好的例子就是数组，比如总的值有10个，取hash值后分别映射到3，8，13，18，22，44，53，63，78，92这10个位置，则我们用一个长度为100的数组可以实现该值域的静态perfect hash。但是你可能会发现有多余的位置并没有被用上，如果能实现长度10的数组完美映射这10个数字，则称之为最小完美hash。动态perfect hash一般比较麻烦，需要做二次hash映射并要第二次映射不会冲突，有兴趣可以查阅相关资料。

**GeoHash**。GeoHash是比较特殊的hash应用，主要是用来快速定位。其原理相对简单（实现起来有不少细节）。主要就是将每一级的地图划分为32块，即每一级用5bit来标识（为啥是5bit，因为最后用base32的编码方式，每个字母或数字5bit），每次缩放一级则用另一个字母或数字标识，最终能得到一串字符串wx4gjk32kfrx，从而一级一级定位直到最小那一级。如划分10级，则最后字符串长度为4，范围到20km，如划分20级，则最后字符串长度为8，范围可以精确到19m。

![img](https://mmbiz.qpic.cn/mmbiz_jpg/JdLkEI9sZfdNbsKXtgzMj9FI5gJFFmvBicDonSl6icibA1Qet1PQQD2VjKS3lWSJeZx4QuxZzGe4Om6zSfoaB9ibTQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

参考https://zhuanlan.zhihu.com/p/35940647