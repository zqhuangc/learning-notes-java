Kibana 是 Elasticsearch 分析和搜索仪表板。

## Kibana 的使用场景

### 实时监控：

通过 histogram 面板，配合不同条件的多个 queries 可以对一个事件走很多个维度组合出不同的时间序列走势。时间序列数据是最常见的监控报警了。

### 问题分析： 

关于 elk 的用途，可以参照其对应的商业产品 splunk 的场景：使用 Splunk 的意义在于使信息收集和处理智能化。而其操作智能化表现在: 
- 搜索，通过下钻数据排查问题，通过分析根本原因来解决问题；  
- 实时可见性，可以将对系统的检测和警报结合在一起，便于跟踪 SLA 和性能问题；  
- 历史分析，可以从中找出趋势和历史模式，行为基线和阈值，生成一致性报告。

### 安装并启动 kibana 

要安装启动 Kibana: 
1. 下载对应平台的 Kibana 二进制包 
2. 解压 .zip 或 tar.gz 压缩文件 
3. 在安装目录里运行: bin/kibana (Linux/MacOSX) 或 bin\kibana.bat (Windows) 
完毕！Kibana 现在运行在 5601 端口了。 

### kibana 连接到 elasticsearch
在config目录下的kibana.yml 文件下 elasticsearch.url: "http://1127.0.0.1:9200"





#### dev tools：

```json
GET _search
{
  "query": {
    "match_all":{}
    "match_phrase": {
      "message" : "nihao"
    }
  }
}
```



