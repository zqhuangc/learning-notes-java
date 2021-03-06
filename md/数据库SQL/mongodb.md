> 摘要: 原创出处 http://www.mongoing.com/archives/3609 「张友东」欢迎转载，保留摘要，谢谢！

------

月初在云栖社区上发起了一个 MongoDB 使用场景及运维管理问题交流探讨 的技术话题，有近5000人关注了该话题讨论，这里就 MongoDB 的使用场景做个简单的总结，谈谈什么场景该用 MongoDB?

很多人比较关心 MongoDB 的适用场景，也有用户在话题里分享了自己的业务场景，比如

案例1

1. 用在应用服务器的日志记录，查找起来比文本灵活，导出也很方便。也是给应用练手，从外围系统开始使用MongoDB。
2. 用在一些第三方信息的获取或者抓取，因为MongoDB的schema-less，所有格式灵活，不用为了各种格式不一样的信息专门设计统一的格式，极大得减少开发的工作。

案例2

1. mongodb之前有用过，主要用来存储一些监控数据，No schema 对开发人员来说，真的很方便，增加字段不用改表结构，而且学习成本极低。

案例3

1. 使用MongoDB做了O2O快递应用，·将送快递骑手、快递商家的信息（包含位置信息）存储在 MongoDB，然后通过 MongoDB 的地理位置查询，这样很方便的实现了查找附近的商家、骑手等功能，使得快递骑手能就近接单，目前在使用MongoDB 上没遇到啥大的问题，官网的文档比较详细，很给力。

经常跟一些同学讨论 MongoDB 业务场景时，会听到类似『你这个场景 mysql 也能解决，没必要一定用 MongoDB』的声音，的确，**并没有某个业务场景必须要使用 MongoDB才能解决，但使用 MongoDB 通常能让你以更低的成本解决问题（包括学习、开发、运维等成本）**，下面是 MongoDB 的主要特性，大家可以对照自己的业务需求看看，匹配的越多，用 MongoDB 就越合适。

| MONGODB 特性            | 优势                                                         |
| ----------------------- | ------------------------------------------------------------ |
| 事务支持                | MongoDB 目前只支持单文档事务，需要复杂事务支持的场景暂时不适合 |
| 灵活的文档模型          | JSON 格式存储最接近真实对象模型，对开发者友好，方便快速开发迭代 |
| 高可用复制集            | 满足数据高可靠、服务高可用的需求，运维简单，故障自动切换     |
| 可扩展分片集群          | 海量数据存储，服务能力水平扩展                               |
| 高性能                  | mmapv1、wiredtiger、mongorocks（rocksdb）、in-memory 等多引擎支持满足各种场景需求 |
| 强大的索引支持          | 地理位置索引可用于构建 各种 O2O 应用、文本索引解决搜索的需求、TTL索引解决历史数据自动过期的需求 |
| Gridfs                  | 解决文件存储的需求                                           |
| aggregation & mapreduce | 解决数据分析场景需求，用户可以自己写查询语句或脚本，将请求都分发到 MongoDB 上完成 |

从目前阿里云 MongoDB 云数据库上的用户看，MongoDB 的应用已经渗透到各个领域，比如**游戏、物流、电商、内容管理、社交、物联网、视频直播**等，以下是几个实际的应用案例。

- 游戏场景，使用 MongoDB 存储游戏用户信息，用户的装备、积分等直接以内嵌文档的形式存储，方便查询、更新
- 物流场景，使用 MongoDB 存储订单信息，订单状态在运送过程中会不断更新，以 MongoDB 内嵌数组的形式来存储，一次查询就能将订单所有的变更读取出来。
- 社交场景，使用 MongoDB 存储存储用户信息，以及用户发表的朋友圈信息，通过地理位置索引实现附近的人、地点等功能
- 物联网场景，使用 MongoDB 存储所有接入的智能设备信息，以及设备汇报的日志信息，并对这些信息进行多维度的分析
- 视频直播，使用 MongoDB 存储用户信息、礼物信息等
- ……

如果你还在为是否应该使用 MongoDB，不如来做几个选择题来辅助决策（注：以下内容改编自 MongoDB 公司 TJ 同学的某次公开技术分享）。

| 应用特征                                           | YES / NO |
| -------------------------------------------------- | -------- |
| 应用不需要事务及复杂 join 支持                     | 必须 Yes |
| 新应用，需求会变，数据模型无法确定，想快速迭代开发 | ？       |
| 应用需要2000-3000以上的读写QPS（更高也可以）       | ？       |
| 应用需要TB甚至 PB 级别数据存储                     | ?        |
| 应用发展迅速，需要能快速水平扩展                   | ?        |
| 应用要求存储的数据不丢失                           | ?        |
| 应用需要99.999%高可用                              | ?        |
| 应用需要大量的地理位置查询、文本查询               | ？       |

如果上述有1个 Yes，可以考虑 MongoDB，2个及以上的 Yes，选择 MongoDB 绝不会后悔。