列存储、文档存储、key-value

MongoDB 将数据存储为一个文档，数据结构由键值(key=>value)对组成。MongoDB 文档类似于` JSON` 对象。字段值可以包含其他文档，数组及文档数组。

mongodb的优势在于轻量化部署非常简单，不用像hbase那样搭一整套hadoop集群，即开即用。hbase更适合离线的海量数据分析



事务                      无事务（性能提升）

表关联                  数据之间没有关联

表结构约束

* mongodb 与 mysql 对比

| SQL术语/概念 | MongoDB术语/概念 | 解释/说明                           |
| ------------ | ---------------- | ----------------------------------- |
| database     | database         | 数据库                              |
| table        | collection       | 数据库表/集合                       |
| row          | document         | 数据记录行/文档                     |
| column       | field            | 数据字段/域                         |
| index        | index            | 索引                                |
| table joins  |                  | 表连接,MongoDB不支持                |
| primary key  | primary key      | 主键,MongoDB自动将_id字段设置为主键 |



增量分析

字段最大并集

MongoDB中的表，字段，不需要预定义

BSON 二进制的 json

网盘数据，分片式存储



* 把索引数据同步到内存，数据存在磁盘

* 不适用场景：

高度事务性

传统的BI

复杂SQL查询



router server

config server

shard server



### 语法

db.yourCollection.crud

原子操作 $set   $xxx

创建数据库：use DATABASE_NAME    切换数据库，没有则创建

删除 db.dropxxx

* 条件查询

```
db.col.find({fieldName : {$gt(条件查询) : 100}})
```

* db.col.find({fieldName : {$type :  2/string}})

| **类型**                | **数字** | **备注**         |
| ----------------------- | -------- | ---------------- |
| Double                  | 1        |                  |
| String                  | 2        |                  |
| Object                  | 3        |                  |
| Array                   | 4        |                  |
| Binary data             | 5        |                  |
| Undefined               | 6        | 已废弃。         |
| Object id               | 7        |                  |
| Boolean                 | 8        |                  |
| Date                    | 9        |                  |
| Null                    | 10       |                  |
| Regular Expression      | 11       |                  |
| JavaScript              | 13       |                  |
| Symbol                  | 14       |                  |
| JavaScript (with scope) | 15       |                  |
| 32-bit integer          | 16       |                  |
| Timestamp               | 17       |                  |
| 64-bit integer          | 18       |                  |
| Min key                 | 255      | Query with `-1`. |
| Max key                 | 127      |                  |



*  db.COLLECTION_NAME.find().limit(NUMBER)  指定数量
* b.COLLECTION_NAME.find(). skip(NUMBER)   跳过数量
* 排序 sort(condition)
* skip(), limilt(), sort()三个放在一起执行的时候，执行的顺序是先 sort(), 然后是 skip()，最后是显示的 limit()。



#### 索引

```
db.collection.createIndex(keys, options)
// options 1 升序  -1  降序
```



聚合：aggregate     类似 count





### 副本

```
rs.initiate({_id: 'conf', members: [{_id: 0, host: 'localhost:27100'}, {_id: 1, host: 'localhost:27101'}]})
```

```
mongod --port PORT --dbpath "YOUR_DB_DATA_PATH" --replSet "REPLICA_SET_INSTANCE_NAME"
启动一个名为 REPLICA_SET_INSTANCE_NAME 的 MongoDB 实例，其端口号为 PORT。

在Mongo客户端使用命令rs.initiate()来启动一个新的副本集。
我们可以使用rs.conf()来查看副本集的配置
查看副本集状态使用 rs.status() 命令
添加：rs.add(HOST_NAME:PORT)


MongoDB中你只能通过主节点将Mongo服务添加到副本集中， 判断当前运行的Mongo服务是否为主节点可以使用命令db.isMaster() 。
MongoDB的副本集与我们常见的主从有所不同，主从在主机宕机后所有服务将停止，而副本集在主机宕机后，副本会接管主节点成为主节点，不会出现宕机的情况。
```



#### 分片

—shardsvr和configsvr参数的作用就是改变启动端口的，所以我们自行指定了端口也可以

还可指定副本集 --replSet =xxx &

```shell
// 端口分布
Shard Server 1：27020
Shard Server 2：27021
Shard Server 3：27022
Shard Server 4：27023
Config Server ：27100
Route Process：40000

// share server
[root@100 /]# mkdir -p /www/mongoDB/shard/s0
[root@100 /]# mkdir -p /www/mongoDB/shard/s1
[root@100 /]# mkdir -p /www/mongoDB/shard/s2
[root@100 /]# mkdir -p /www/mongoDB/shard/s3
[root@100 /]# mkdir -p /www/mongoDB/shard/log
[root@100 /]# /usr/local/mongoDB/bin/mongod --port 27020 --dbpath=/www/mongoDB/shard/s0 --logpath=/www/mongoDB/shard/log/s0.log --logappend --fork
....
[root@100 /]# /usr/local/mongoDB/bin/mongod --port 27023 --dbpath=/www/mongoDB/shard/s3 --logpath=/www/mongoDB/shard/log/s3.log --logappend --fork

// config server
[root@100 /]# mkdir -p /www/mongoDB/shard/config
[root@100 /]# /usr/local/mongoDB/bin/mongod --port 27100 --dbpath=/www/mongoDB/shard/config --logpath=/www/mongoDB/shard/log/config.log --logappend --fork

// route process
/usr/local/mongoDB/bin/mongos --port 40000 --configdb localhost:27100 --fork --logpath=/www/mongoDB/shard/log/route.log --chunkSize 500

// 配置Sharding
[root@100 shard]# /usr/local/mongoDB/bin/mongo admin（库名） --port 40000
MongoDB shell version: 2.0.7
connecting to: 127.0.0.1:40000/admin
mongos> db.runCommand({ addshard:"localhost:27020" })
{ "shardAdded" : "shard0000", "ok" : 1 }
......
mongos> db.runCommand({ addshard:"localhost:27029" })
{ "shardAdded" : "shard0009", "ok" : 1 }
mongos> db.runCommand({ enablesharding:"test" }) #设置分片存储的数据库
{ "ok" : 1 }
mongos> db.runCommand({ shardcollection: "test.log", key: { id:1,time:1}})
{ "collectionsharded" : "test.log", "ok" : 1 }
```

#### 备份与恢复

```shell
mongodump -h dbhost -d dbname -o dbdirectory
-h：
MongDB所在服务器地址，例如：127.0.0.1，当然也可以指定端口号：127.0.0.1:27017

-d：
需要备份的数据库实例，例如：test

-o：
备份的数据存放位置，例如：c:\data\dump，当然该目录需要提前建立，在备份完成后，系统自动在dump目录下建立一个test目录，这个目录里面存放该数据库实例的备份数据。


mongorestore -h <hostname><:port> -d dbname <path>
--host <:port>, -h <:port>：
MongoDB所在服务器地址，默认为： localhost:27017

--db , -d ：
需要恢复的数据库实例，例如：test，当然这个名称也可以和备份时候的不一样，比如test2

--drop：
恢复的时候，先删除当前数据，然后恢复备份的数据。就是说，恢复后，备份后添加修改的数据都会被删除，慎用哦！

<path>：
mongorestore 最后的一个参数，设置备份数据所在位置，例如：c:\data\dump\test。

你不能同时指定 <path> 和 --dir 选项，--dir也可以设置备份目录。

--dir：
指定备份的目录

你不能同时指定 <path> 和 --dir 选项。
```

监控：mongostat  mongotop 



### Mongo-java api

```java
import java.net.UnknownHostException;
 
import com.mongodb.BasicDBObject;
import com.mongodb.DB;
import com.mongodb.DBCollection;
import com.mongodb.DBCursor;
import com.mongodb.DBObject;
import com.mongodb.Mongo;
import com.mongodb.MongoException;
 
public class MongoDb_Test {
 
	public static void main(String[] args) {
 
		try {
			// 实例化Mongo对象，连接27017端口
			Mongo mongo = new Mongo("localhost", 27017);
			// 连接名为yourdb的数据库，假如数据库不存在的话，mongodb会自动建立
			DB db = mongo.getDB("yourdb");
			// Get collection from MongoDB, database named "yourDB"
			// 从Mongodb中获得名为yourColleection的数据集合，如果该数据集合不存在，Mongodb会为其新建立
			DBCollection collection = db.getCollection("yourCollection");
			// 使用BasicDBObject对象创建一个mongodb的document,并给予赋值。
			BasicDBObject document = new BasicDBObject();
			document.put("id", 1001);
			document.put("msg", "hello world mongoDB in Java");
			// 将新建立的document保存到collection中去
			collection.insert(document);
			// 创建要查询的document
			BasicDBObject searchQuery = new BasicDBObject();
			searchQuery.put("id", 1001);
			// 使用collection的find方法查找document
			DBCursor cursor = collection.find(searchQuery);
			// 循环输出结果
			while (cursor.hasNext()) {
				System.out.println(cursor.next());
			}
			System.out.println("Done");
		} catch (UnknownHostException e) {
			e.printStackTrace();
		} catch (MongoException e) {
			e.printStackTrace();
		}
	}
}

```

###  morphia

```java
public class DatastoreTest {
    private Mongo mongo;
    private Morphia morphia;
    private Datastore ds;
    
    public void init() {
        mongo = new Mongo();//Mongoclient
        morphia = new Morphia();
        morphia.map(User.class);
        ds = morphia.createDatastore(mongo, "temp");
        Iterable<User> it = ds.createQuery(User.class).fetch();
        while(it.iterator().hasNext()) {
            print("fetch: " + it.iterator().next());
        }
    }

    
```

### spring-data-mongo

xml 配置文件

```java
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:context="http://www.springframework.org/schema/context"
	xmlns:mongo="http://www.springframework.org/schema/data/mongo"
	xsi:schemaLocation="
	http://www.springframework.org/schema/data/mongo 
	http://www.springframework.org/schema/data/mongo/spring-mongo.xsd
	http://www.springframework.org/schema/beans 
	http://www.springframework.org/schema/beans/spring-beans.xsd
	http://www.springframework.org/schema/context  
    http://www.springframework.org/schema/context/spring-context.xsd">
 
	<context:component-scan base-package="cn.slimsmart.mongodb.demo.spring" />
	<context:property-placeholder location="classpath*:mongodb.properties" />
 
	<mongo:mongo id="mongo" host="${monngo.host}" port="${monngo.port}">
		<mongo:options connections-per-host="8"
			threads-allowed-to-block-for-connection-multiplier="4"
			connect-timeout="1000" max-wait-time="1500" auto-connect-retry="true"
			socket-keep-alive="true" socket-timeout="1500" slave-ok="true"
			write-number="1" write-timeout="0" write-fsync="true" />
	</mongo:mongo>
	<!-- mongo的工厂，通过它来取得mongo实例,dbname为mongodb的数据库名，没有的话会自动创建 -->
	<mongo:db-factory id="mongoDbFactory" dbname="User"
		mongo-ref="mongo" />
 
	<!-- mongodb的主要操作对象，所有对mongodb的增删改查的操作都是通过它完成 -->
	<bean id="mongoTemplate" class="org.springframework.data.mongodb.core.MongoTemplate">
		<constructor-arg name="mongoDbFactory" ref="mongoDbFactory" />
	</bean>
 
	<!-- 映射转换器，扫描back-package目录下的文件，根据注释，把它们作为mongodb的一个collection的映射 User -->
	<mongo:mapping-converter base-package="cn.slimsmart.mongodb.demo.spring" />
 
	<!-- mongodb bean的仓库目录，会自动扫描扩展了MongoRepository接口的接口进行注入 -->
	<mongo:repositories base-package="cn.slimsmart.mongodb.demo.spring" />
	 <!-- To translate any MongoExceptions thrown in @Repository annotated classes --> 
	<context:annotation-config />
 
</beans>

```





实体 @Document

@Resource







#### 特性

1.可以通过@Configuration注解或者XML风格配置
2.MongoTemplate 辅助类 (类似JdbcTemplate),方便常用的CRUD操作
3.异常转换
4.丰富的对象映射
5.通过注解指定对象映射
6.持久化和映射声明周期事件
7.通过MongoReader/MongoWriter 定义底层的映射
8.基于Java的Query, Criteria, Update DSL
9.自动实现Repository，可以提供定制的查找
10.QueryDSL 支持类型安全的查询
11.跨数据库平台的持久化 - 支持JPA with Mongo
12.GeoSpatial 支持
13.Map-Reduce 支持
14.JMX管理和监控
15.CDI 支持
16.GridFS 支持





## MongoDB高级应用

用户创建

db.help

类似json





### 架构

routeserver,configserver,replica,shard,chunk

####  主从

dbpath=/path1、logpath=/path2、logappend=true、fork=true

配置 master=true，source=slave_ip:slave_port

配置 slave=true，source=master_ip:master_port

#### 主从仲裁

[mongodb副本集](https://www.cnblogs.com/zhoujinyi/p/3554010.html)

replset=shard001(副本集，要一样)



```
replSet info you may need to run replSetInitiate -- rs.initiate() in the shell -- if that is not already done

rs.initiate({"_id":"mmm","members":[
{"_id":1,"host":"192.168.200.252:27017","priority":1},
{"_id":2,"host":"192.168.200.245:27017","priority":1}
]})

rs.status()
######
"_id": 副本集的名称
"members": 副本集的服务器列表
"_id": 服务器的唯一ID
"host": 服务器主机
"priority": 是优先级，默认为1，优先级0为被动节点，不能成为活跃节点。优先级不位0则按照由大到小选出活跃节点。
"arbiterOnly": 仲裁节点，只参与投票，不接收数据，也不能成为活跃节点。


rs.add("192.168.200.25:27017")
rs.remove("192.168.200.25:27017")
```

**副本集中数据同步过程**：Primary节点写入数据，Secondary通过读取Primary的oplog得到复制信息，开始复制数据并且将复制信息写入到自己的oplog。

注意：在副本集的环境中，要是所有的Secondary都宕机了，只剩下Primary。最后Primary会变成Secondary，不能提供服务。



#### 一主多从

随机选举，去中心化，不设定优先级，不设仲裁

```
rs.initiate({"_id":"mmm","members":[
{"_id":1,"host":"192.168.200.252:27017"},
{"_id":2,"host":"192.168.200.245:27017"}
{"_id":3,"host":"192.168.200.264:27017"}
]})
```



### 分片集群

分片路由

mongos -help 

* config server

```conf
dbpath=/path1、logpath=/path2、logappend=true、fork=true
configsvr=true、port=28001、replset=configs
bind_ip=ip

rs.initiate({"_id":"configs","members":[
{"_id":1,"host":"192.168.200.252:28001"},
{"_id":2,"host":"192.168.200.245:28001"}
{"_id":3,"host":"192.168.200.264:28001"}
]})
```



* router server

```conf
configdb=configs/192.168.200.252:28001,192.168.200.245:28001,192.168.200.264:28001
logpath=/path2
logappend=true
fork=true
port=30000
bind_ip=ip
```

* shard server

chunk

```conf
sh.addShard("shard1/10.0.0.152:27017");
sh.enabkeSharding(db);
sh.shardCollection("数据库名称.集合名称",key : {分片键: 1}  )

####
db.runCommand( { addshard : "sh1/10.0.0.152:28021,10.0.0.152:28022,10.0.0.152:28023",name:"shard1"} )
db.runCommand( { addshard : "sh2/10.0.0.152:28024,10.0.0.152:28025,10.0.0.152:28026",name:"shard2"} )
```

--shardsvr true







| : "joe" } })          | SELECT * FROM users WHERE username <> 'joe';                 |                                                              |
| --------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| IN 操作符             | db.raffle.find({ "ticket_no" : { "$in" : [725, 542, 390] } })    $in非常灵活，可以指定不同类型 的条件和值。  例如在逐步将用户的ID号迁移成用户名的过程中，   查询时需要同时匹配ID和用户名 | SELECT ticket_no FROM raffles WHERE ticket_no IN (725, 542, 390); |
| NOT IN 操作符         | db.raffle.find({ "ticket_no" : { "$nin" : [725, 542, 390] } }) | SELECT * FROM raffles WHERE ticket_no not in (725, 542, 390); |
| OR 操作符             | db.raffle.find({ "$or" : [{ "ticket_no" : 725 }, { "winner" : true }] }) | SELECT * FROM raffles WHERE ticket_no = 725 OR winner = 'true'; |
| 空值检查              | db.c.find({"y" : null}) null不仅会匹配某个键的值为null的文档  ，而且还会匹配不包含这个键的文档。  所以，这种匹配还会返回缺少这个键的所有文档。  如果 仅想要匹配键值为null的文档，  既要检查改建的值是否为null,   还要通过 $exists 条件 判定键值已经存在   db.c.find({ "z" : { "$in" : [null], "$exists" : true }}) | SELECT * FROM cs WHERE z is null;                            |
| 多列排序              | db.c.find().sort({ username : 1, age: -1 })                  | SELECT * FROM cs ORDER BY username ASC, age DESC;            |
| AND操作符             | db.users.find({ "$and" : [{ "x" : { "$lt" : 1 }, { "x" : 4 } }] })   由于查询优化器不会对 $and进行优化，  所以可以改写成下面的 db.users.find({ "x" : { "$lt" : 1, "$in" : [4] } }) | SELECT * FROM users WHERE x > 1 AND x IN (4);                |
| NOT 操作符            | db.users.find({ "id_num" : { "$not" : { "$mod" : [5,1] } } }) | SELECT * FROM users WHERE id_num NOT IN (5,1);               |
| LIKE 操作符(正则匹配) | db.blogs.find( {  "title" : /post?/i } )    MongoDB 使用Perl兼容的正则表达式(PCRE) 库来匹配正则表达式，  任何PCRE支持表达式的正则表达式语法都能被MongoDB接受 | SELECT * FROM blogs WHERE title LIKE "post%";                |

## 五、 函数对比

```json
{ "_id" : 1, "item" : "abc", "price" : 10, "quantity" : 2 }
{ "_id" : 2, "item" : "jkl", "price" : 20, "quantity" : 1 }
{ "_id" : 3, "item" : "xyz", "price" : 5, "quantity" : 5 }
{ "_id" : 4, "item" : "abc", "price" : 10, "quantity" : 10 }
{ "_id" : 5, "item" : "xyz", "price" : 5, "quantity" : 10 }
```

| 函数对比 | MongoDB                                                      | MySQL                             |
| -------- | ------------------------------------------------------------ | --------------------------------- |
| COUNT    | db.foo.count()                                               | SELECT COUNT(id) FROM foo;        |
| DISTINCT | db.runCommand({ "distinct": "people", "key": "age" })        | SELECT DISTINCT(age) FROM people; |
| MIN      | db.sales.aggregate( [ { $group: { _id: {}, minQuantity: { $min: "$quantity" } } } ]);     结果： { "_id" : {  }, "minQuantity" : 1 } | SELECT MIN(quantity) FROM sales;  |
| MAX      | db.sales.aggregate( [ { $group: { _id: {}, maxQuantity: { $max: "$quantity" } } } ]); | SELECT MAX(quantity) FROM sales;  |
| AVG      | db.sales.aggregate( [ { $group: { _id: {}, avgQuantity: { $avg: "$quantity" } } } ]); | SELECT AVG(quantity) FROM sales;  |
| SUM      | db.sales.aggregate( [ { $group: { _id: {}, totalPrice: { $sum: "$price" } } } ]); | SELECT SUM(price) FROM sales;     |

## 六、 CURD 对比

| CURD 对比      | MongoDB                                                      | MySQL                                                        |
| -------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 插入数据       | post = {"title" : "My Blog Post", "content" : "Here`s my blog post"}; db.blog.insert(post) 如果blog 这个集合不存在，则会创建 | INSERT INTO blogs(`title`, `blog_content`) VALUES ('My Blog Post', 'Here`s my blog post.') |
| 批量插入       | db.blog.batchInsert([{ "title" : "AAA", "content" : "AAA---" }, { "title" : "BBB", "content" : "JJJJ--" }])   当前版本的MongoDB能接受最大消息长度48MB，   所以在一次批量插入中能插入的文档是有限制的。  并且在执行批量插入的过程中，有一个文档插入失败，  那么在这个文档之前的所有文档都会成功插入到集合中，  而这个文档以及之后的所有文档全部插入失败。 | INSERT INTO blogs(`title`, `blog_content`) VALUES('AAA', 'AAA---'), ('BBB', 'BBB---'); |
| 查询数据       | db.blog.find(); db.blog.findOne();                           | SELECT * FROM blogs;   SELECT * FROM blogs LIMIT 1;          |
| 更新旧数据     | post.blog_content = "十一";   db.blog.update({title: "My Blog Post"}, post) | UPDATE set blog_content = "十一" WHERE title = "My Blog Post"; |
| 更新新增COLUMN | post.comments = "very good";   db.blog.update({title : "My Blog Post"}, post) | ALTER table `blogs` ADD COLUMN comments varchar(200);   UPDATE `blogs` set comments = "very good" WHERE title = 'My Blog Post'; |
| 删除数据       | db.blog.remove({ title : "My Blog Post" })                   | DELETE FROM `blogs` WHERE title = 'My Blog Post'             |
| 校验           | post.blog_visit = 123;    db.blog.update({title : "My Blog Post"}, post);   post.blog_visit = "asd.123aaa";    db.blog.update({title : "My Blog Post"}, post)    插入的时候，检查大小。所有的文档都必须小于16MB。  这样做的目的是为了防止不良的模式设计，并且保持性能一直。由于MongoDB只进行最基本的检查，所以插入非法的数据很容易。 | 类型校验，长度校验。   ALTER table `blogs` ADD COLUMN blog_visit INT(10);    UPDATE blogs SET blog_visit = "asdasd" WHERE id = 1;   ERROR 1366 (HY000): Incorrect integer value: 'asdasd' for column 'blog_visit' at row 1 |
| 删除表         | db.blog.remove({}), db.blog.drop()                           | DELETE from blogs; drop table blogs;                         |

## 七、有的没的

**MongoDB:**

- GridFS

  可以用来存储大文件(>16M), 与MySQL BLOB，TEXT 类似。

- MapReduce

  MapReduce
   是一种计算模型，简单的说就是将大批量的工作数据分解执行，然后再将结果合并成
   最终结果。MongoDB提供的MapReduce
   非常灵活，对于大规模数据分析也相当实用。

- 时间有限的集合

  MongoDB 2.2 引入一个新特性--
   TTL集合，TTL集合支持失效时间设置，使用expireAfterSeconds 来实现
   当超过指定时间后，集合自动清除超时的文档，这用来保存一些诸如session会话信息
   的时候非常有用，或者存储数据使用。

- 无JOIN

  MongoDB 为了更快的读，以及更方便的分布式，抛弃了JOIN操作。
   JOIN开销其实很大。

- 关于锁

  当资源被代码的多个部分所共享时，需要确定这处资源只能在一个地方被操作。
   就版本的MongoDB(pre
   2.0)拥有一个全局的写入锁。这就意味着贯穿整个服务器只有一个地方做写操作。
   这就可能导致数据库因为某个地方锁定超负载而停滞。这个问题在2.0版本中得到
   了显著的改善，并且在2.2版本中得到了进一步的加强。MongoDB
   2.2使用数据库级别的锁再这个问题上迈进了一大步。

  两个不同版本的MongoDB，写入性能对比。



  MySQL InnoDB使用行级锁，有效提高并发。

- 无事务

  不像MySQL
   这些多行数据原子操作的传统数据库。MongoDB只支持单个文件的原子修改。
   但这也正是MongoDB可以更快地读的原因，没有事务这些负载的处理。MongoDB
   可以轻松处理TB级别的数据。

- 磁盘消耗

  MongoDB
   会消耗太多的磁盘空间了。当然，这与它的编码方式有关，因为MongoDB会通过预分配
   大文件空间来避免磁盘碎片问题。它的工作方式是这样：在创建数据库时，系统会创建
   一个名为[db_name].0的文件，当该文件有一半以上被使用时，系统会再次创建一个名
   为[db_namel].1的文件，该文件的大小是方才的两倍。这个情况会持续不断的发生，因此
   256、512、1024、2048大小的文件会被写到磁盘上。

**MySQL:**

- 强大的引擎

  InnoDB 引擎： MySQL默认的事务性引擎。它被设计用来处理大量的短期事务，短期
   事务大部分情况是正常提交的，很少会被回滚。InnoDB采用MVCC来支持高并发。

  MyISAM 引擎： 5.1以及之前的默认版本。全文索引，压缩，空间函数，但是MyISAM不支持事务和
   行级锁。MyISAM最整张表加锁，而不是针对行。读取时会对需要读到的所有表加
   共享锁，写入时则对表加排它锁。

  Memory 引擎：比MyISAM表块一个数量级，因为所有的数据都保存在内存中，不需要进行磁盘I/O。数据会丢失

  Infobright 是最有名的面向列的存储引擎。但该引擎不支持索引。

- 事务

  确保数据库的状态从一个一致状态转变为另一个一致状态。一致状态的含义是数据库中

  的数据应满足完整性约束。

- schema

  有利于数据整理，数据存储，并执行正规化的行为。 保证数据的完整性，一致性。

  描述了数据存储的模板，比如创建table。

  校验数据的格式，比如整形的column 就不能存放字符串数据。

## 八、 MySQL 与 MongoDB 写入对比

| options                                           | MySQL                       | MongoDB                     |
| ------------------------------------------------- | --------------------------- | --------------------------- |
| Time taken for tests:                             | 548.281 seconds             | 661.318 seconds             |
| Total transferred:                                | 44000000 bytes              | 44200000 bytes              |
| Requests per second:                              | 182.39 [#/sec]              | 151.21 [#/sec]              |
| Time per request:                                 | 274.141 [ms]                | 330.659 [ms]                |
| Time per request(across all concurrent requests): | 5.483 [ms]                  | 6.613 [ms]                  |
| Transfer rate:                                    | 78.37 [Kbytes/sec] received | 65.27 [Kbytes/sec] received |

在本测试例子中，MySQL 写入情况好于 MongoDB

压测参考命令：

```swift
   ab -c 50 -n 100000 http://127.0.0.1:6666/deals/mysql_write
   ab -c 50 -n 100000 http://127.0.0.1:6666/deals/mongodb_write
```

## 九、 MySQL 与 MongoDB 读取对比

| options                                           | MySQL                        | MongoDB                     |
| ------------------------------------------------- | ---------------------------- | --------------------------- |
| Time taken for tests:                             | 1181.881 seconds             | 606.406 seconds             |
| Failed requests:                                  | 2239                         | 0                           |
| Non-2xx responses:                                | 2239                         | 0                           |
| Total transferred:                                | 359397490 bytes              | 44100000 bytes              |
| Requests per second:                              | 84.61 [#/sec]                | 164.91 [#/sec]              |
| Time per request:                                 | 590.941 [ms]                 | 303.203 [ms]                |
| Time per request(across all concurrent requests): | 11.819 [ms]                  | 6.064 [ms]                  |
| Transfer rate:                                    | 296.96 [Kbytes/sec] received | 71.02 [Kbytes/sec] received |

在本测试例子中， MySQL 读取性能没有 MongoDB好

压测参考命令:

```swift
   ab -c 50 -n 100000 http://127.0.0.1:6666/deals/mysql_read
   ab -c 50 -n 100000 http://127.0.0.1:6666/deals/mongodb_read
```

