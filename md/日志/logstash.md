```
Logstash 通过管道进行运作，管道有两个必需的元素，输入和输出，还有一个可选的元素，过滤器。

输入插件从数据源获取数据，过滤器插件根据用户指定的数据格式修改数据，输出插件则将数据写入到目的地。
```


Logstash是一个接收，处理，转发日志的工具。支持系统日志，webserver日志，错误日志，应用日志，总之包括所有可以抛出来的日志类型。怎么样听起来挺厉害的吧？
在一个典型的使用场景下(ELK)：用Elasticsearch作为后台数据的存储，kibana用来前端的报表展示。Logstash在其过程中担任搬运工的角色，它为数据存储，报表查询和日志解析创建了一个功能强大的管道链。Logstash提供了多种多样的 input,filters,codecs和output组件，让使用者轻松实现强大的功能。好了让我们开始吧
```bash
bin/logstash -e 'input { stdin { } } output { elasticsearch { host => localhost } }'

curl 'http://localhost:9200/_search?pretty'
```

Elasticsearch-kopf插件
多重输出
默认配置 - 按照每日日期建立索引


## logstash一些核心的特性和如何和logstash引擎交互

### 事件的生命周期
Inputs,Outputs,Codecs,Filters构成了Logstash的核心配置项。Logstash通过建立一条事件处理的管道，从你的日志提取出数据保存到Elasticsearch中，为高效的查询数据提供基础。

### Inputs

input 及输入是指日志数据传输到Logstash中。其中常见的配置如下：
* file：从文件系统中读取一个文件，很像UNIX命令 "tail -0a"
* syslog：监听514端口，按照RFC3164标准解析日志数据
* redis：从redis服务器读取数据，支持channel(发布订阅)和list模式。redis一般在Logstash消费集群中作为"broker"角色，保存events队列共Logstash消费。
* lumberjack：使用lumberjack协议来接收数据，目前已经改为 logstash-forwarder。

### Filters

Fillters 在Logstash处理链中担任中间处理组件。他们经常被组合起来实现一些特定的行为来，处理匹配特定规则的事件流。常见的filters如下：  
* grok：解析无规则的文字并转化为有结构的格式。Grok 是目前最好的方式来将无结构的数据转换为有结构可查询的数据。有120多种匹配规则，会有一种满足你的需要。
* mutate：mutate filter 允许改变输入的文档，你可以从命名，删除，移动或者修改字段在处理事件的过程中。
* drop：丢弃一部分events不进行处理，例如：debug events。
* clone：拷贝 event，这个过程中也可以添加或移除字段。
* geoip：添加地理信息(为前台kibana图形化展示使用)

### Outputs

outputs是logstash处理管道的最末端组件。一个event可以在处理过程中经过多重输出，但是一旦所有的outputs都执行结束，这个event也就完成生命周期。一些常用的outputs包括：
elasticsearch：如果你计划将高效的保存数据，并且能够方便和简单的进行查询
* file：将event数据保存到文件中。
* graphite：将event数据发送到图形化组件中，一个很流行的开源存储图形化展示的组件。http://graphite.wikidot.com/。
* statsd：statsd是一个统计服务，比如技术和时间统计，通过udp通讯，聚合一个或者多个后台服务，如果你已经开始使用statsd，该选项对你应该很有用。

### Codecs

* codecs 是基于数据流的过滤器，它可以作为input，output的一部分配置。Codecs可以帮助你轻松的分割发送过来已经被序列化的数据。流行的codecs包括 json,msgpack,plain(text)。
* json：使用json格式对数据进行编码/解码
multiline：将汇多个事件中数据汇总为一个单一的行。比如：java异常信息和堆栈信息
获取完整的配置信息，请参考 Logstash文档中 "plugin configuration"部分。

### 使用配置文件
使用-e参数在命令行中指定配置是很常用的方式
```
bin/logstash -f logstash-simple.conf
```

实用的例子

Apache 日志(从文件获取)
条件判断
Syslog