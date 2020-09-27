[官方文档](https://www.elastic.co/guide/index.html)

核心组成
ELK由Elasticsearch、Logstash和Kibana三部分组件组成；

* Elasticsearch是个开源分布式搜索引擎，它的特点有：分布式，零配置，自动发现，索引自动分片，索引副本机制，restful风格接口，多数据源，自动搜索负载等。
* Logstash是一个完全开源的工具，它可以对你的日志进行收集、分析，并将其存储供以后使用
* kibana 是一个开源和免费的工具，它可以为 Logstash 和 ElasticSearch 提供的日志分析友好的 Web 界面，可以帮助您汇总、分析和搜索重要数据日志。

elasticsearch：后台分布式存储以及全文检索 
logstash: 日志加工、“搬运工”  [logstash文档](https://doc.yonyoucloud.com/doc/logstash-best-practice-cn/input/file.html)
kibana：数据可视化展示。
Beats
Beats分为很多种，每一种收集特定的信息。常用的是Filebeat，监听文件变化，传送文件内容。一般日志系统使用Filebeat就够了。
